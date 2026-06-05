---
name: migrate-selenium-java-to-codeceptjs
description: "Port a Selenium WebDriver or Selenide (Java) test suite to CodeceptJS 4. Trigger when the project contains `pom.xml` declaring `<artifactId>selenium-java</artifactId>` / `<artifactId>selenide</artifactId>` / `<artifactId>webdrivermanager</artifactId>`, `build.gradle` / `build.gradle.kts` with `org.seleniumhq.selenium:selenium-java` or `com.codeborne:selenide`, a `src/test/java/` tree with `*Test.java` / `*IT.java` / `*Tests.java` / `*Steps.java`, imports from `org.openqa.selenium.*` (`WebDriver`, `WebElement`, `By`, `WebDriverWait`, `ExpectedConditions`, `Actions`, `JavascriptExecutor`, `Select`, `Keys`) or `com.codeborne.selenide.*` (`Selenide`, `SelenideElement`, `Configuration`, `Condition`, `ElementsCollection`), JUnit 5 / 4 or TestNG annotations (`@Test`, `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@BeforeMethod`, `@BeforeClass`, `@DataProvider`, `@ParameterizedTest`, `@FindBy`, `@FindAll`), `PageFactory.initElements(...)`, `WebDriverManager.*.setup()`, `new ChromeDriver(...)` / `new RemoteWebDriver(...)`, `Selenide.open(...)`, `$(...).shouldBe(...)` / `$$(...).filter(...)` chains, or Cucumber-JVM step defs (`@Given` / `@When` / `@Then` from `io.cucumber.java.en.*`). Walks the port end-to-end: inventory page objects (almost always present, often `@FindBy`-driven), shared `*Helper` / `*Manager` classes, JUnit/TestNG hooks and listeners, data providers, Cucumber step defs; install CodeceptJS in a parallel directory with the **WebDriver helper as the default** (most native target — same W3C protocol the Java suite already speaks); no driver setup is needed because WebdriverIO v9 auto-starts the matching browser driver, with **Docker Selenium (Selenoid / `selenium/standalone-chrome`) only as a fallback** for parallel/CI/Grid runs per `codeceptjs-fundamentals` and `node_modules/codeceptjs/docs/webdriver.md § \"Selenium in Docker (Selenoid)\"`; port the config (Maven/Gradle deps → `package.json`, Selenide `Configuration.*` / Selenium capabilities → `helpers.WebDriver.*`); split shared Java helpers into `WebExtra` (browser-driven — `JavascriptExecutor` bodies become `browser.execute(...)` via `this.helpers['WebDriver'].browser`) and `ApiExtras` (RestAssured / Apache HttpClient → REST helper, never via the browser helper); port `@FindBy`-driven page objects to CodeceptJS page objects without inventing assertion/one-liner wrappers; **drop explicit wait code** (`WebDriverWait`, `ExpectedConditions.*`, Selenide `.shouldBe(visible)` / `.shouldHave(text(...))` chains) — CodeceptJS auto-waits via `smartWait`; translate `By.X(...)` / `@FindBy(...)` locators to semantic strings / ARIA / `locate()` / `{ css }`; convert specs (handing off to `writing-codeceptjs-tests`), mapping JUnit 5 / JUnit 4 / TestNG annotations to `Feature` / `Scenario` / `Before` / `BeforeSuite` / `After` / `AfterSuite` and `@DataProvider` / `@ParameterizedTest` to `Data(...).Scenario(...)`; swap any custom `LoginHelper` / cookie-based session reuse for the `auth` plugin; keep WireMock as a sidecar for client-side stubs while on WebDriver (WebDriver has no native network interception); for Cucumber-JVM, keep `.feature` files and rewrite step defs in JS via CodeceptJS BDD; then decommission the Java suite. Once the WebDriver suite is green, propose the **optional Playwright swap** (faster, cross-browser from one config, native `I.mockRoute`) — mechanical config change, test code unchanged."
---

# Migrate Selenium / Selenide (Java) → CodeceptJS 4

This migration is more than a syntax port — every file changes language. Java becomes JavaScript (or TypeScript), Maven/Gradle becomes `npm`, JUnit/TestNG becomes the CodeceptJS runner. The mechanical translation is fully tractable; the wins are real (semantic locators, auto-wait, first-class abstractions, AI-assisted authoring), and the patterns transfer.

Three foundational differences to internalize:

1. **Tests read sequentially without explicit waits.** CodeceptJS auto-queues every `I.*` call onto an internal recorder and waits on DOM/element stability automatically. Selenium's `WebDriverWait` + `ExpectedConditions.*` boilerplate goes to zero; Selenide's `.shouldBe(visible)` / `.shouldHave(text(...))` chains usually collapse into the action itself (`I.click('Save')` waits, clicks, asserts). `await` in CodeceptJS is reserved for grabs (`await I.grabTextFrom(...)`) — on plain actions it works but isn't recommended. **This is the single biggest visible change in spec files.**
2. **Helpers, not `WebDriver` / `driver`.** `I.*` dispatches to a configured helper. **Default to the WebDriver helper** — it speaks the same W3C WebDriver protocol your Java suite already speaks, so the migration is most native: same Selenium server, same browser drivers, same capabilities, same Grid if you have one. Once the suite is green on WebDriver, you can swap in the **Playwright** helper (modern, faster, less flaky, native multi-browser via one config) by changing one helper block — the test code is identical either way.
3. **First-class abstractions.** Page objects, the `auth` plugin, multi-user `session(...)`, custom helpers, and the `customLocator` plugin are built in. Java suites already centralise these (`PageFactory`, `LoginHelper`, `DriverManager`); the migration consolidates them onto the framework's idioms instead of carrying the Java-specific glue forward.

Authoritative references: `node_modules/codeceptjs/docs/basics.md`, `locators.md`, `playwright.md`, `webdriver.md`, `custom-helpers.md`, `pageobjects.md`.

## When to trigger

Any of:

- `pom.xml` declaring `selenium-java`, `selenide`, `webdrivermanager`, `junit-jupiter`, `junit`, or `testng` as a dependency.
- `build.gradle` / `build.gradle.kts` with `org.seleniumhq.selenium:selenium-java`, `com.codeborne:selenide`, `org.junit.jupiter:junit-jupiter`, or `org.testng:testng`.
- A `src/test/java/` tree with `*Test.java` / `*IT.java` / `*Tests.java` / `*Steps.java`.
- Imports from `org.openqa.selenium.*` (`WebDriver`, `WebElement`, `By`, `WebDriverWait`, `ExpectedConditions`, `Actions`, `JavascriptExecutor`, `Select`, `Keys`).
- Imports from `com.codeborne.selenide.*` (`Selenide`, `SelenideElement`, `Configuration`, `Condition`, `ElementsCollection`).
- JUnit / TestNG annotations: `@Test`, `@BeforeEach` / `@BeforeMethod` / `@BeforeClass` / `@BeforeAll` / `@BeforeSuite`, `@After*`, `@DataProvider`, `@ParameterizedTest`, `@FindBy` / `@FindAll`.
- Code calls `PageFactory.initElements(...)`, `WebDriverManager.*.setup()`, `new ChromeDriver(...)`, `new RemoteWebDriver(...)`, `Selenide.open(...)`, `$(...)` / `$$(...)` chains.
- Cucumber-JVM step definitions (`io.cucumber.java.en.*`) under `src/test/java/`.
- The user says "migrate / port / convert from Selenium / Selenide / Java".

## What does not migrate

Be honest up-front:

- **Java language features in test code** — generics, streams, lambdas, custom exception hierarchies, AspectJ weaving, JUnit `@ExtendWith` extensions, TestNG `IInvokedMethodListener` listeners. These re-express as plain JS — usually shorter, but pick the simplest port, not a faithful translation.
- **Maven / Gradle build phases tied to tests** — pre/post integration phases, profile-driven test exclusions, surefire-failsafe split. Re-implement with npm scripts and CodeceptJS `--grep` / `tag` filters.
- **Client-side network interception** under the default WebDriver helper. WebDriver has no equivalent to Playwright's `I.mockRoute(...)`. While on WebDriver, keep WireMock (or any proxy / service-virtualisation tool) as a sidecar. After the optional Playwright swap, port stubs to `I.mockRoute(...)` if desired.
- **Custom Selenium `EventFiringWebDriver` / Selenide event listeners** — CodeceptJS uses event hooks (`event.test.before`, `event.step.failed`) and the `aiTrace` plugin instead; reimplement against those if a listener was load-bearing.
- **Java-specific reporting (Allure annotations on test methods, ExtentReports, ReportPortal Java agents)** — switch to a CodeceptJS reporter. `@testomatio/reporter` is the most direct equivalent for dashboards; Allure has a CodeceptJS adapter for teams that must keep Allure formats.
- **Cucumber-JVM step-def language and Hooks** — feature files port as-is, step defs are rewritten in JS via CodeceptJS BDD. Java-specific glue (`@ScenarioScope`, PicoContainer DI) does not.

## Workflow

Run phases in order. Commit at each boundary so any regression is bisectable.

### 1. Inventory the Java project

Before touching anything, build a picture. Two passes.

**Shape of the project** — grep / `find` for cost predictors:

- `pom.xml` / `build.gradle{,.kts}` — which Selenium/Selenide/JUnit/TestNG versions, which extra deps (RestAssured, WireMock, Allure, Cucumber, AssertJ, Hamcrest, Awaitility).
- `src/test/java/**/*.java` — file count, package layout.
- Driver bootstrap — `WebDriverManager.*.setup()`, `new ChromeDriver(...)`, `RemoteWebDriver`, `Configuration.*` (Selenide) usage sites; usually centralised in a `DriverManager` / `BrowserFactory` / `BaseTest` class.
- Count occurrences of `@FindBy(`, `@FindAll(`, `PageFactory.initElements(`, `WebDriverWait(`, `ExpectedConditions.`, `.shouldBe(`, `.shouldHave(`, `.shouldNot(`, `JavascriptExecutor`, `new Actions(`, `new Select(`, `@DataProvider`, `@ParameterizedTest`, `Cucumber.feature` — each maps to a known replacement pattern.

**Shared logic and page objects** — Java suites are POM-heavy and helper-heavy. Find them before touching tests:

- **Page objects** — typically under `src/test/java/**/pages/`, `**/pageobjects/`, `**/po/`. Classes with `@FindBy` fields + methods that drive interactions. **Almost always present.** Port one-to-one to CodeceptJS page objects.
- **Base test classes** — `BaseTest`, `BaseUITest`, `AbstractTest`. Usually hold `@BeforeEach` driver setup, `@AfterEach` driver teardown, screenshot-on-failure logic, and shared utilities. Driver setup goes away (CodeceptJS owns the lifecycle); screenshot-on-failure becomes the `screenshot` plugin; shared utilities split into helpers (see below).
- **Helper / Manager classes** — `LoginHelper`, `ApiHelper`, `DbHelper`, `BrowserHelper`, `DataFactory`, `WaitUtils`. UI helpers (DOM tricks, JS execution, file uploads via Robot) go to `WebExtra`; HTTP helpers (RestAssured wrappers, programmatic login, seed/teardown via API) go to `ApiExtras`; DB and pure utility classes become plain JS modules or `ApiExtras` methods.
- **Listeners / extensions** — TestNG `ITestListener`, JUnit 5 `@ExtendWith(...)`. Become CodeceptJS plugins, event hooks, or `Before` / `After` hooks.
- **Data providers / fixtures** — `@DataProvider` methods, `@ParameterizedTest` sources, JSON/CSV/Excel files loaded via Jackson / OpenCSV / Apache POI. Become `Data(...)` for CodeceptJS data-driven scenarios and ES imports / `fs/promises` loads.
- **Cucumber step defs** — if present, list every step regex and its current Java body. Each will become a JS step def using CodeceptJS BDD.

Produce a short inventory: every shared abstraction with its current Java location and planned CodeceptJS destination. The user reviews before any code is written.

### 2. Install CodeceptJS in a parallel directory

Cross-language migrations need parallel coexistence — you can't edit Java files into JS in place. Create a sibling directory (e.g. `e2e/`, `codeceptjs/`, or a separate repo if the team prefers a clean split):

```bash
mkdir e2e && cd e2e
npx codeceptjs init
```

**Pick the WebDriver helper.** This is the native target for a Selenium-Java port — same W3C protocol, same capabilities, and the same Grid if you have one. The Java suite keeps running through CI in parallel, so coverage stays green until the port is complete.

**No driver setup needed by default.** The WebDriver helper runs on WebdriverIO v9, which auto-downloads and starts the matching browser driver on the fly. You do **not** install Chromedriver / Geckodriver, run a Selenium server, or set `host` / `port`. Just configure the helper and run — nothing new is required:

```js
// codecept.conf.js
helpers: {
  WebDriver: {
    url: 'https://your-app.example.com',
    browser: 'chrome',
    windowSize: '1280x720',
    smartWait: 5000,
    desiredCapabilities: {
      'goog:chromeOptions': { args: ['--disable-gpu', '--no-sandbox'] },
    },
  },
},
```

The helper starts a local driver only when no connection info (`host` / `port`) is set — so leaving them out is what enables auto-management.

`smartWait` gives you Selenide-equivalent implicit waiting on every locator lookup; with auto-retries on top, almost every `WebDriverWait` / `ExpectedConditions.*` line from the Java suite disappears.

**Fallback: run Selenium in Docker.** Only needed when you want parallel/isolated runs, a pinned browser version, CI without a local browser, or an existing Grid — the convention `codeceptjs-fundamentals` and `node_modules/codeceptjs/docs/webdriver.md § "Selenium in Docker (Selenoid)"` describe. Start a container and add `host` / `port` to the helper block (which then disables auto-management and points at the container instead):

```bash
# Selenoid (parallel-friendly) — see https://aerokube.com/selenoid for full setup
docker run -d --name selenoid -p 4444:4444 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/selenoid:/etc/selenoid aerokube/selenoid:latest-release

# or the simplest single-browser option
docker run -d --name selenium -p 4444:4444 --shm-size=2g \
  selenium/standalone-chrome:latest
```

```js
// add to the WebDriver block only when using the Docker/Grid fallback
host: '127.0.0.1',
port: 4444,
```

**Smoke-test the install before any porting.** `npx codeceptjs run --steps` with a stub Scenario (`Scenario('boot', ({ I }) => I.amOnPage('/'))`) must open a browser and pass. Fix driver / capability / container issues here, not after you've ported abstractions.

**After the suite is green: consider Playwright.** Once the migration is complete and tests are stable on WebDriver, swap the helper to gain cross-browser coverage (Chromium / Firefox / WebKit) from one config, `I.mockRoute(...)` network mocking, and roughly 2-3× faster runs. The migration is mechanical — replace the `WebDriver` helper block with a `Playwright` block (see `playwright.md`), stop the Docker fallback if you were using it, rerun. Test code stays identical. **Don't do this until WebDriver-green** — debugging two new variables at once (helper + ported tests) is wasted effort.

### 3. Port the config

Map Java config sources → `codecept.conf.{js,ts}`. **Primary target: `WebDriver` helper** (Playwright alternates shown for the later swap):

| Java source | CodeceptJS 4 (`WebDriver` helper — default) | Playwright (later swap) |
|---|---|---|
| `Configuration.baseUrl` (Selenide) | `helpers.WebDriver.url` | `helpers.Playwright.url` |
| `Configuration.browser = "chrome"` | `helpers.WebDriver.browser: 'chrome'` | `helpers.Playwright.browser: 'chromium'` |
| `Configuration.headless = true` | inject `--headless` into `desiredCapabilities['goog:chromeOptions'].args` (or use `setHeadlessWhen(CI)`) | `helpers.Playwright.show: false` |
| `Configuration.timeout` / Selenium implicit waits | `helpers.WebDriver.smartWait` (auto-retry on every locator lookup) | `helpers.Playwright.waitForTimeout` |
| `pageLoadTimeout` | `helpers.WebDriver.timeouts['page load']` | `helpers.Playwright.timeout` |
| `Configuration.browserSize = "1280x720"` | `helpers.WebDriver.windowSize: '1280x720'` | `helpers.Playwright.windowSize: '1280x720'` |
| `ChromeOptions().addArguments(...)` | `desiredCapabilities['goog:chromeOptions'].args` | `chromium.args` / `launchOptions.args` |
| `WebDriverManager.*.setup()` | drop — WebdriverIO v9 auto-manages the driver (no setup); Docker only if using the fallback | drop — Playwright manages browsers |
| `RemoteWebDriver` URL → Selenium Grid | add `helpers.WebDriver.host` / `port` to point at the Grid / Selenoid (fallback path) | drop Grid — Playwright runs locally |
| Maven `surefire` / `failsafe` `<systemPropertyVariables>` | `process.env.*` | `process.env.*` |
| `pom.xml` / `build.gradle` test deps | `package.json` `devDependencies` (`codeceptjs`, `webdriverio`, optional plugins) | `package.json` (`codeceptjs`, `playwright`) |
| `@Test(retryAnalyzer = ...)` / Selenide retry | top-level `retry: N` | top-level `retry: N` |
| `@Test(timeOut = 30000)` (TestNG) | per-Scenario `Scenario(...).timeout(30)` | per-Scenario `Scenario(...).timeout(30)` |
| Allure on test methods | `@testomatio/reporter` or the Allure CodeceptJS adapter | same |

### 4. Port shared abstractions

This is the bedrock. Do it before any spec rewrite — every spec rewrite shrinks because the verbs it needs (`I.doSmth(...)`, `loginPage.fill(...)`) already exist.

**Hard rule for shared helper classes.** Every method on a Java `*Helper` / `*Util` / `*Manager` class becomes a method on a custom CodeceptJS helper. **Split across two helpers by the kind of operation** — they have different access patterns and different correct APIs:

- **`WebExtra`** (`lib/helpers/WebExtra.js`) for **browser-driven** operations — anything that needs the open page, DOM, JS execution, storage, network-response waits. **This is where every `JavascriptExecutor.executeScript(...)` body lands**, as `browser.execute(...)` (WebDriver) or `page.evaluate(...)` (Playwright, after the later swap). Reaches `this.helpers['WebDriver'].browser` — a WebdriverIO browser instance. See `node_modules/codeceptjs/docs/custom-helpers.md § WebDriver Example`.
- **`ApiExtras`** (`lib/helpers/ApiExtras.js`) for **pure HTTP** operations — programmatic login, seed/teardown data, CRUD against an API. **This is where every RestAssured wrapper / Apache HttpClient call lands**, routed through the REST helper. Reaches `this.helpers['REST']` (or `GraphQL`). See `node_modules/codeceptjs/docs/api.md`.

Register both helpers under `helpers` in `codecept.conf.{js,ts}`.

**Never bypass the REST helper for API work** by reaching for the browser helper's underlying HTTP surface. That skips the REST + `JSONResponse` stack — no step logging, no `I.seeResponseCodeIsSuccessful` assertions, no shared headers. If the API needs the same auth as the browser, share cookies once at the top of the config:

```js
import { setSharedCookies } from '@codeceptjs/configure'
setSharedCookies()
```

…or set `defaultHeaders` on the REST helper for token-based auth, or use `I.amBearerAuthenticated(secret(token))` per test.

**WebExtra example** — browser-driven against the WebDriver helper (a `JavascriptExecutor` call from a `BrowserHelper.java` lands here as `browser.execute(...)`):

```js
import Helper from '@codeceptjs/helper'
import fs from 'node:fs/promises'

export default class WebExtra extends Helper {
  async scrollIntoView(selector) {
    const { browser } = this.helpers['WebDriver']
    const el = await browser.$(selector)
    await el.scrollIntoView()
  }

  async setLocalStorage(key, value) {
    const { browser } = this.helpers['WebDriver']
    await browser.execute((k, v) => localStorage.setItem(k, v), key, value)
  }

  async writeJsonFile(filePath, data) {
    await fs.writeFile(filePath, JSON.stringify(data, null, 2))
  }
}
```

> When you later swap to the Playwright helper, only the body of each method changes — the `I.*` surface contributed by `WebExtra` stays the same. Replace `this.helpers['WebDriver'].browser` with `this.helpers['Playwright'].page` and translate `browser.execute(...)` to `page.evaluate(...)`. Tests don't change.

**ApiExtras example** — pure HTTP (a RestAssured wrapper from `LoginHelper.java` lands here):

```js
import Helper from '@codeceptjs/helper'

export default class ApiExtras extends Helper {
  async loginViaApi(email, password) {
    const REST = this.helpers['REST']
    await REST.sendPostRequest('/api/auth/login', { email, password })
  }

  async seedUser(payload) {
    const REST = this.helpers['REST']
    const { data } = await REST.sendPostRequest('/api/users', payload)
    return data
  }
}
```

**Helper code style** — applies to both:

- All `import` statements at the **top of the file**. Never `const fs = await import('node:fs/promises')` inside a method.
- Use built-in assertions (`I.seeResponseCodeIsSuccessful` for API, `I.seeElement` for browser), `ExpectHelper`, or factories from `codeceptjs/assertions` — **never** `if (cond) throw new Error('...')`. Failures must render as proper assertion errors.
- If your `WebExtra` is growing a session-cache map keyed by user, you are reimplementing the `auth` plugin — stop and let the `auth` plugin (phase 8) handle session reuse.

**Page objects.** Java POMs port across very cleanly because CodeceptJS treats page objects as first-class DI citizens. **Port conservatively** — keep only the methods the original Java class had. Drop `PageFactory` entirely; the lazy lookup `@FindBy` provided is exactly what plain locator strings in `I.*` calls already do.

```java
// Java
public class LoginPage {
  @FindBy(id = "email")    WebElement emailField;
  @FindBy(css = "[type=submit]") WebElement submitBtn;

  public LoginPage(WebDriver driver) { PageFactory.initElements(driver, this); }

  public void login(String email, String password) {
    emailField.sendKeys(email);
    $(By.id("password")).setValue(password);
    submitBtn.click();
  }
}
```

```js
// CodeceptJS
import { inject } from 'codeceptjs'

const { I } = inject()

export default {
  fields: {
    email: '#email',
    password: '#password',
    submit: { css: '[type=submit]' },
  },

  login(email, password) {
    I.fillField(this.fields.email, email)
    I.fillField(this.fields.password, secret(password))
    I.click(this.fields.submit)
  },
}
```

Register under `include` in `codecept.conf.{js,ts}` so the page object auto-injects into Scenarios.

Page-object anti-patterns to avoid (unless the original Java code already had them):

- **Assertion methods** (`checkTitleVisible() { I.seeElement(...) }`) — page objects are action verbs (`fillForm`, `submitOrder`); let assertions live in the test.
- **One-liner wrappers** around a single `I.click` / `I.see*` / `I.grabTextFrom` — the wrapper buys nothing over calling `I.*` from the test.
- **Methods used by only one test** — leave the steps in the test.
- **`if (cond) throw new Error(...)`** in any method — use `I.see*`, `I.seeNumberOfElements`, `ExpectHelper`, or `codeceptjs/assertions` factories instead.

**Hooks.** Java `@BeforeEach` / `@BeforeMethod` driver setup goes away (CodeceptJS owns the lifecycle). Other hook bodies port to CodeceptJS hooks: `@BeforeEach` / `@BeforeMethod` → `Before(({ I }) => ...)`, `@BeforeAll` / `@BeforeClass` → `BeforeSuite(...)`, `@After*` → matching `After*`. Project-wide setup (DB seed, API token mint) goes in `bootstrap()` / `teardown()` in config.

Sanity-check before moving on: `npx codeceptjs check -c <config>` must pass, and `npx codeceptjs list -c <config>` must show every Java helper method as an `I.*` action contributed by `WebExtra` or `ApiExtras` — whichever owns it.

### 5. Convert spec files

One file at a time, leaning on the abstractions from phase 4. Hand off the per-spec work to the **`writing-codeceptjs-tests`** skill — it drives the live browser via MCP and verifies each step before committing.

| Java (JUnit 5 / TestNG / Selenide) | CodeceptJS 4 |
|---|---|
| `class LoginTest { @Test void X() { ... } }` | `Feature('Login')` + `Scenario('X', ({ I }) => { ... })` |
| `@Test`, multiple per class | one `Scenario(...)` each |
| `@DisplayName("X")` (JUnit 5) / `@Test(description = "X")` (TestNG) | the `Scenario(...)` title |
| `@BeforeEach` / `@BeforeMethod` | `Before(({ I }) => { ... })` |
| `@AfterEach` / `@AfterMethod` | `After(({ I }) => { ... })` |
| `@BeforeAll` / `@BeforeClass` | `BeforeSuite(({ I }) => { ... })` |
| `@AfterAll` / `@AfterClass` | `AfterSuite(...)` |
| `@Disabled` / `@Ignore` / `enabled = false` | `xScenario(...)` or `Scenario.skip(...)` |
| `@Tag("smoke")` (JUnit 5) / `@Test(groups = "smoke")` (TestNG) | `Scenario(...).tag('@smoke')` |
| `@ParameterizedTest` + `@CsvSource(...)` | `Data([{ ... }, { ... }]).Scenario(...)` |
| `@Test(dataProvider = "rows")` + `@DataProvider` | same: `Data(rows).Scenario(...)` |
| `driver.get('/x')` / `Selenide.open('/x')` | `I.amOnPage('/x')` |
| `loginPage.login(...)` (POM call) | `LoginPage.login(...)` (page object auto-injected) |

**Drop explicit wait code.** This is the biggest mechanical edit:

```java
// Selenium — gone
new WebDriverWait(driver, Duration.ofSeconds(10))
    .until(ExpectedConditions.elementToBeClickable(By.id("save")))
        .click();

// Selenide — collapses
$("#save").shouldBe(visible).click();
```

```js
// CodeceptJS — auto-waits
I.click('#save')
```

If a specific wait is load-bearing (waiting for a spinner to disappear before clicking), keep it with the right `I.waitFor*`:

```js
I.waitForInvisible('.spinner', 10)
I.click('#save')
```

**Iteration.** In tests, page objects, and helpers, use **`for...of`** for any loop containing `I.*` calls. Never `Array.prototype.forEach`. `.forEach` swallows the iteration callback's return — an `await` inside it does not block the outer function, and the CodeceptJS recorder may queue steps out of order or finish the Scenario before the loop is done.

```js
for (const sort of testSort) {
  I.click(locate(this.filterFormLabel).withText(sort))
}
```

**Dry-run as you go.** After each batch of converted specs, run:

```bash
npx codeceptjs dry-run --steps -c <config>
```

It loads every scenario, resolves every `I.*` call against the configured helpers, and prints the step list — all without launching a browser. Typos, missing imports, page objects not registered under `include`, and `I.*` verbs that don't exist on `WebExtra` / `ApiExtras` all surface here in seconds. Fix anything that fails before running a real test.

**Then run the whole batch for real.** Dry-run proves specs parse and resolve — not that they pass. As soon as a batch is dry-run clean, run it against the browser:

```bash
npx codeceptjs run --steps -c <config>
```

First real runs after a migration almost always have failures — locator drift, timing the explicit `WebDriverWait` code was masking, auth/session differences, data assumptions. **This is expected; fixing it is part of the migration, not a follow-up task.** When a test fails, **invoke the `debugging-codeceptjs-tests` skill and fix it on the fly** — it breakpoints the failing step, inspects the live page via MCP, finds the working locator/wait, and commits the verified fix. Do not bulk-rewrite specs blind, and do not mask failures with `retry`. Drive every failure to a real fix before starting the next batch. A batch is "done" when it runs green, not when it dry-runs clean.

### 6. Locators

CodeceptJS priority — pick the highest that fits:

1. **Semantic strings** — button text, label, placeholder, link text: `I.click('Save')`, `I.fillField('Email', 'u@t.com')`.
2. **ARIA roles** — `I.click({ role: 'button', name: 'Sign In' })`.
3. **`locate()` builder** — `I.click(locate('button').withText('Edit').inside('tr').withText('Acme'))`.
4. **CSS / XPath** — fallback only.

Java suites lean on `By.id`, `By.cssSelector`, `By.xpath`, `By.linkText`, and `@FindBy(...)` heavily. Translation:

| Java | CodeceptJS |
|---|---|
| `By.id("save")` / `@FindBy(id = "save")` | `'#save'` |
| `By.cssSelector(...)` / `@FindBy(css = ...)` | `{ css: '...' }` |
| `By.xpath(...)` / `@FindBy(xpath = ...)` | `{ xpath: '...' }` |
| `By.linkText("Edit")` / `@FindBy(linkText = ...)` | `'Edit'` (semantic) |
| `By.partialLinkText("Edit")` | `locate('a').withText('Edit')` |
| `By.tagName("button")` / `By.className(...)` | `'button'` / `'.cls'` |
| `By.name("email")` | `{ name: 'email' }` (Selenide-equivalent `[name=email]` lookup) |
| `@FindBy(how = How.NAME, using = "email")` | `{ name: 'email' }` |
| `Selenide $(...).find(...)` / `.parent()` | `locate(...).inside(...)` / `.parent()` |
| `$$(...).filter(text("Active")).first()` | `locate(...).withText('Active').first()` |

If the suite uses many `[data-test=...]` / `[data-qa=...]` attributes, enable the `customLocator` plugin so they read as `$submit` instead of `{ css: '[data-test=submit]' }`. Full guidance in **`writing-codeceptjs-tests`** § Locators.

### 7. Actions, assertions, grabs

| Java (Selenium / Selenide) | CodeceptJS 4 |
|---|---|
| `el.click()` / `$(...).click()` | `I.click(sel)` |
| `el.sendKeys("x")` / `$(...).setValue("x")` | `I.fillField(sel, 'x')` |
| `el.clear()` / `$(...).clear()` | `I.clearField(sel)` |
| `new Select(el).selectByVisibleText("A")` / `$(...).selectOption("A")` | `I.selectOption(sel, 'A')` |
| Checkbox `el.click()` / `$(...).setSelected(true)` | `I.checkOption(sel)` / `I.uncheckOption(sel)` |
| `new Actions(driver).moveToElement(el).perform()` | `I.moveCursorTo(sel)` |
| `Actions...dragAndDrop(a, b).perform()` | `I.dragAndDrop(srcSel, dstSel)` |
| `Actions...keyDown(Keys.CONTROL).sendKeys("a")...` | `I.pressKey(['Control', 'a'])` |
| `el.isDisplayed()` / `$(...).shouldBe(visible)` | `I.seeElement(sel)` |
| `el.getText()` / `$(...).getText()` | `await I.grabTextFrom(sel)` |
| `$(...).shouldHave(text("X"))` / `assertEquals("X", el.getText())` | `I.see('X', sel)` |
| `$(...).shouldHave(value("X"))` / `assertEquals("X", el.getAttribute("value"))` | `I.seeInField(sel, 'X')` |
| `$$(...).shouldHave(size(5))` / `assertEquals(5, els.size())` | `I.seeNumberOfElements(sel, 5)` |
| `driver.getCurrentUrl().contains("/x")` / `webdriver().shouldHave(url(...))` | `I.seeInCurrentUrl('/x')` |
| `((JavascriptExecutor) driver).executeScript(js, args)` | method on `WebExtra` calling `page.evaluate` |
| `driver.manage().getCookies()` | `await I.grabCookie()` |
| `driver.manage().addCookie(...)` | `I.setCookie({...})` |
| `((TakesScreenshot) driver).getScreenshotAs(...)` | drop — `screenshot` plugin handles it on failure; `I.saveScreenshot('name.png')` for manual |

`await` only on grabs. Plain actions queue automatically.

### 8. Sessions and auth

Java suites almost always have a `LoginHelper` / `AuthBase` that performs UI login or programmatic login + cookie injection. Both port to the **`auth` plugin**. Hand off to the **`codeceptjs-auth`** skill for the setup walk-through.

- Programmatic login → method on `ApiExtras` (`loginViaApi`); the `auth` plugin's role definition calls it once and caches the resulting cookies / storage state.
- UI login → method on `WebExtra` (`login`); the `auth` plugin's role calls it the same way.
- Cookie reuse across tests → drop the manual logic; `auth` caches cookies + storage from the helper (Playwright `storageState`, WebDriver cookie jar — same plugin config either way).
- Multi-user scenarios → `session(...)` from `codeceptjs/effects`.

### 9. Test data and fixtures

| Java | CodeceptJS 4 |
|---|---|
| `@DataProvider` returning `Object[][]` | `Data([{ ... }, { ... }]).Scenario(...)` |
| `@ParameterizedTest` + `@CsvSource` / `@ValueSource` | `Data([...]).Scenario(...)` |
| Jackson `ObjectMapper.readValue(jsonFile, ...)` | `import users from './fixtures/users.json' with { type: 'json' }` |
| OpenCSV `CSVReader` | `@codeceptjs/data` or plain `csv-parse` import |
| Apache POI Excel reads | rare; convert sheets to JSON/CSV ahead of time or use `xlsx` if truly needed |
| `Faker` (Java) | `@faker-js/faker` (drop-in) |

### 10. API testing and mocking

| Java | CodeceptJS 4 |
|---|---|
| RestAssured `given().when().then()` | REST helper: `await I.sendPostRequest(...)` + `I.seeResponseCodeIsSuccessful` / `I.seeResponseContainsKeys` / Zod schema |
| Apache HttpClient wrappers | method on `ApiExtras` via REST helper |
| WireMock client-side stubs | **on WebDriver: keep WireMock** (or any sidecar proxy) — WebDriver has no client-side network interception. After the later Playwright swap, port to `I.mockRoute(url, route => route.fulfill({...}))` |
| WireMock backend sidecar | **keep WireMock** running as a sidecar process; CodeceptJS does not replace backend service mocking either way |

> Client-side network mocking is the one capability the WebDriver helper does not match. If the Java suite relies heavily on WireMock client-side stubs and you want them inline as `I.mockRoute(...)`, that's a strong reason to schedule the Playwright swap right after WebDriver-green.

### 11. Cucumber-JVM (if present)

If the Java suite is Cucumber-driven:

- `.feature` files stay where they are (or move to `features/` under the CodeceptJS project).
- Configure CodeceptJS BDD (`node_modules/codeceptjs/docs/bdd.md`) to load them.
- Rewrite each Java step def as a JS step def, calling `I.*` and page objects. The step text and regex stay identical so the features don't change.
- Java-specific Cucumber glue (`@ScenarioScope`, PicoContainer, `cucumber.options`) drops — CodeceptJS BDD has its own hook surface.

### 12. Decommission the Java suite

Only after every spec is ported and CI is green on the CodeceptJS run:

- Drop the Java test sources (`src/test/java/`, `src/test/resources/` if test-only).
- Remove Selenium / Selenide / JUnit / TestNG / RestAssured / WireMock-Java / Allure / `webdrivermanager` from `pom.xml` / `build.gradle`.
- Remove the Maven / Gradle test invocations from CI; replace with `npx codeceptjs run-workers <N>` (or `run`) jobs.

## Verify

1. Default path: nothing to check — WebdriverIO v9 starts the driver. Only if using the Docker fallback: container reachable, `curl -fsS http://127.0.0.1:4444/status` returns `ready: true`.
2. `npx codeceptjs check -c <config>` — config + helper + plugin sanity.
3. `npx codeceptjs list -c <config>` — every ported Java helper method appears as an `I.*` action from `WebExtra` or `ApiExtras`; every page object's methods appear.
4. `npx codeceptjs dry-run --steps -c <config>` — every Scenario loads.
5. Full run on WebDriver: `npx codeceptjs run --steps -c <config>`. Failures are expected on first runs — drive each to a fix via the **`debugging-codeceptjs-tests`** skill (not `retry`, not blind rewrites). The migration is complete only when the whole converted suite is green.
6. Hand off to **`codeceptjs-run-analysis`** to inspect `output/trace_*/` artifacts (requires the `aiTrace` plugin enabled).
7. `find src/test/java -name "*.java"` — empty before deleting the Java tree.

## Optional follow-up: swap to Playwright

Once the migration is complete and WebDriver runs are stable in CI, consider swapping the helper for ongoing work:

1. `npm i -D playwright`.
2. Replace the `WebDriver` block under `helpers` with a `Playwright` block (see `node_modules/codeceptjs/docs/playwright.md`); drop `host` / `port` / `desiredCapabilities`.
3. Translate each `WebExtra` method body from `this.helpers['WebDriver'].browser.execute(...)` to `this.helpers['Playwright'].page.evaluate(...)`. The `I.*` surface the helper contributes is unchanged.
4. If you were using the Docker fallback, stop the container — Playwright manages browsers itself.
5. Rerun `dry-run` and the smoke suite. Page objects, specs, fixtures, and the `auth` plugin config don't change.

You gain: cross-browser coverage (Chromium / Firefox / WebKit) from one config, faster runs, native `I.mockRoute(...)` network mocking, richer ARIA snapshots via the MCP loop.

## Pointers

- `node_modules/codeceptjs/docs/basics.md` — `I.*` vocabulary, locators, assertions, the `await` rule
- `node_modules/codeceptjs/docs/webdriver.md` — default helper for this migration; § "Selenium in Docker (Selenoid)" for the container setup
- `node_modules/codeceptjs/docs/playwright.md` — target for the optional follow-up swap; `mockRoute` for WireMock client-side stubs
- `node_modules/codeceptjs/docs/locators.md` — semantic / ARIA / `locate()`, `customLocator` plugin
- `node_modules/codeceptjs/docs/pageobjects.md` — for ported `@FindBy` POMs
- `node_modules/codeceptjs/docs/custom-helpers.md` — `WebExtra` / `ApiExtras` patterns (see § "WebDriver Example" for the `browser` access pattern)
- `node_modules/codeceptjs/docs/api.md` — REST helper for RestAssured ports
- `node_modules/codeceptjs/docs/auth.md` — replace `LoginHelper` + cookie reuse
- `node_modules/codeceptjs/docs/bdd.md` — CodeceptJS BDD for Cucumber-JVM ports
- `node_modules/codeceptjs/docs/effects.md` — `session`, `tryTo`, `retryTo`, `within`
- **`codeceptjs-fundamentals`** — Docker-fallback Selenium setup convention; run **after** migration to confirm wiring
- **`writing-codeceptjs-tests`** — per-spec rewrite playbook (drive via MCP, learn locators, commit verified steps)
- **`debugging-codeceptjs-tests`** — **use on every failing test from the first full run** (breakpoint, inspect live page via MCP, fix on the fly)
- **`codeceptjs-auth`** — replace `LoginHelper` / cookie-based session reuse
