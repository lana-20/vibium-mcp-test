---
name: vibium-mcp-test
description: Regression test suite for 9 known vibium MCP tool bugs (MB1–MB9), ordered by priority and severity (P1 Critical first, P3 Low last). Run after fixes to verify each bug is resolved. Labels PASS/FAIL/SKIP with exact repro steps and cross-site hardening.
---

# vibium MCP Regression Test Suite

Run all 9 tests and produce a final summary table. Each test maps to a bug observed during MCP tool exercise sessions. Tests are ordered by priority — MB1–MB4 are P1, MB5–MB7 are P2, MB8–MB9 are P3.

## Setup

Ensure the browser is running before starting:

```
browser_start {}
browser_navigate { url: "https://testtrack.org" }
browser_get_title {}
```

If `browser_get_title` returns an error, stop and restart:
```
browser_stop {}
browser_start {}
```

---

## Tests

Print a result line for each test:
- `PASS MB<n>` — bug is fixed
- `FAIL MB<n>` — bug still present (include exact error or symptom)
- `SKIP MB<n>` — test could not run (explain why)

---

### MB1 — `browser_count` — Go type mismatch crash (Critical · P1)

```
browser_navigate { url: "https://testtrack.org" }
browser_count { selector: "a" }
```

PASS if: output is an integer (e.g. `42`)
FAIL if: `json: cannot unmarshal number into Go struct field .result.result.value of type string`

Workaround — use `browser_evaluate` with `.toString()` to force string return:
```
browser_evaluate { expression: "document.querySelectorAll('a').length.toString()" }
```

PASS (workaround) if: returns a string like `"42"` without error

**Cross-site hardening** — test on different frontend stacks:

```
# Coffee Cart — Vue SPA
browser_navigate { url: "https://coffee-cart.app/" }
browser_count { selector: "button" }
```
PASS if: integer; FAIL if: unmarshal error

```
# Swag Labs — React app (requires login first)
browser_navigate { url: "https://www.saucedemo.com" }
browser_fill { selector: "#user-name", text: "standard_user" }
browser_fill { selector: "#password", text: "secret_sauce" }
browser_click { selector: "#login-button" }
browser_count { selector: "button" }
```
PASS if: integer; FAIL if: unmarshal error

```
# AcademyBugs — vanilla JS, bug-planted site
browser_navigate { url: "https://academybugs.com/" }
browser_count { selector: "a" }
```
PASS if: integer; FAIL if: unmarshal error

```
# The Internet — multi-example vanilla JS site
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_count { selector: "li a" }
```
PASS if: integer; FAIL if: unmarshal error

```
# Automation Exercise — Angular-like e-commerce
browser_navigate { url: "https://www.automationexercise.com/products" }
browser_count { selector: "button" }
```
PASS if: integer; FAIL if: unmarshal error

```
# Practice Software Testing — e-commerce site
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_count { selector: "a" }
```
PASS if: integer; FAIL if: unmarshal error

```
# DemoQA — component library site
browser_navigate { url: "https://demoqa.com/" }
browser_count { selector: "div.card" }
```
PASS if: integer; FAIL if: unmarshal error

PASS (cross-site) if: integer returned on all 7 sites
FAIL (cross-site) if: unmarshal error on any site

---

### MB2 — `browser_storage_state` — cookie object parse error (Critical · P1)

```
browser_navigate { url: "https://testtrack.org" }
browser_storage_state {}
```

PASS if: JSON state returned with cookies array, localStorage, sessionStorage
FAIL if: `failed to get cookies: failed to parse storage.getCookies result: json: cannot unmarshal object into Go struct field Cookie.cookies.value of type string`

Note: `browser_restore_storage` cannot be meaningfully tested until MB2 is fixed — no valid state file can be produced.

**Cross-site hardening** — test on cookie-rich sites:

```
# Saucedemo — sets session cookie after login
browser_navigate { url: "https://www.saucedemo.com" }
browser_fill { selector: "#user-name", text: "standard_user" }
browser_fill { selector: "#password", text: "secret_sauce" }
browser_click { selector: "#login-button" }
browser_storage_state {}
```
PASS if: cookies array contains session cookie; FAIL if: unmarshal error

```
# Coffee Cart — Vue SPA with cookies
browser_navigate { url: "https://coffee-cart.app/" }
browser_storage_state {}
```
PASS if: JSON state returned; FAIL if: unmarshal error

```
# AcademyBugs — sets cookie banner / analytics cookies
browser_navigate { url: "https://academybugs.com/" }
browser_storage_state {}
```
PASS if: JSON state returned; FAIL if: unmarshal error

```
# The Internet — Heroku app with cookies
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_storage_state {}
```
PASS if: JSON state returned; FAIL if: unmarshal error

```
# Practice Software Testing — e-commerce with session cookies
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_storage_state {}
```
PASS if: JSON state returned; FAIL if: unmarshal error

```
# DemoQA — sets analytics/preference cookies
browser_navigate { url: "https://demoqa.com/" }
browser_storage_state {}
```
PASS if: JSON state returned; FAIL if: unmarshal error

PASS (cross-site) if: JSON state returned on all 6 sites
FAIL (cross-site) if: unmarshal error on any site

---

### MB3 — `browser_dialog_accept` / `browser_dialog_dismiss` — deadlock with `browser_click` (Critical · P1)

**Caution:** if the direct-click test FAILS, the browser session will be deadlocked. Use `browser_stop {}` then `browser_start {}` to recover. Allow ~30 seconds before manually quitting if the call does not return.

**Note:** direct click deadlock is confirmed on at least 4 sites: The Internet, Evil Tester, testautomationpractice.blogspot.com, and testtrack.org/alert-demo. It is a BiDi-level bug, not site-specific.

**Direct click test (expected deadlock):**
```
browser_navigate { url: "https://the-internet.herokuapp.com/javascript_alerts" }
browser_find { role: "button", text: "Click for JS Alert" }
browser_click { selector: "@e1" }
browser_dialog_accept {}
```

PASS if: `browser_click` returns and `browser_dialog_accept` succeeds — deadlock is fixed
FAIL if: `browser_click` hangs indefinitely — restart browser session before continuing

**Workaround verification** — setTimeout approach (must work regardless of deadlock fix status):
```
browser_navigate { url: "https://the-internet.herokuapp.com/javascript_alerts" }
browser_evaluate { expression: "setTimeout(() => alert('test alert'), 300)" }
browser_dialog_accept {}
```
PASS (workaround) if: `Dialog accepted`
FAIL (workaround broken) if: `no such alert` or browser hangs

```
browser_evaluate { expression: "setTimeout(() => confirm('test confirm'), 300)" }
browser_dialog_dismiss {}
```
PASS if: `Dialog dismissed`

```
browser_evaluate { expression: "setTimeout(() => window.__r = prompt('enter:'), 300)" }
browser_dialog_accept { text: "vibium" }
```
PASS if: `Dialog accepted with text: "vibium"`

**Cross-site hardening** — sites known to trigger native JS dialogs:

```
# Evil Tester alert page — multiple dialog types
browser_navigate { url: "https://testpages.eviltester.com/styled/alerts/alert-test.html" }
browser_evaluate { expression: "setTimeout(() => alert('evil tester test'), 300)" }
browser_dialog_accept {}
```
PASS if: `Dialog accepted`; FAIL if: error or deadlock

```
# Verify pre-stub workaround on Evil Tester (B3 pre-stub pattern from CLI suite)
browser_navigate { url: "https://testpages.eviltester.com/styled/alerts/alert-test.html" }
browser_evaluate { expression: "window.alert = () => {}; window.confirm = () => true; window.prompt = () => 'vibium-test'" }
browser_find { role: "button", text: "Show alert box" }
browser_click { selector: "@e1" }
```
PASS if: click returns immediately (pre-stub intercepts native dialog)
FAIL if: click hangs despite pre-stub

```
# Automation Testing Practice — alert buttons
browser_navigate { url: "https://testautomationpractice.blogspot.com/" }
browser_evaluate { expression: "setTimeout(() => confirm('blogspot confirm'), 300)" }
browser_dialog_dismiss {}
```
PASS if: `Dialog dismissed`; FAIL if: error or deadlock

```
# DemoQA alerts page — alert and confirm dialogs
browser_navigate { url: "https://demoqa.com/alerts" }
browser_evaluate { expression: "setTimeout(() => alert('demoqa test'), 300)" }
browser_dialog_accept {}
```
PASS if: `Dialog accepted`; FAIL if: error or deadlock

```
# testtrack.org alert demo — all three dialog types
browser_navigate { url: "https://testtrack.org/alert-demo" }
browser_evaluate { expression: "setTimeout(() => alert('testtrack alert test'), 300)" }
browser_dialog_accept {}
browser_evaluate { expression: "setTimeout(() => confirm('testtrack confirm test'), 300)" }
browser_dialog_dismiss {}
browser_evaluate { expression: "setTimeout(() => prompt('testtrack prompt test'), 300)" }
browser_dialog_accept { text: "vibium" }
```
PASS if: all three dialogs handled; FAIL if: error or deadlock

```
# Practice Test Automation — login page (no dialog, but verify dialog tools don't break normal pages)
browser_navigate { url: "https://practicetestautomation.com/practice-test-login/" }
browser_dialog_accept {}
```
PASS if: `no such alert` error (expected — no dialog present, tool correctly reports absence)
FAIL if: browser crashes or session breaks

PASS (cross-site) if: workaround works on all 5 dialog sites; accept/dismiss return correctly on non-dialog site
FAIL (cross-site) if: workaround fails or browser deadlocks on any site

---

### MB4 — `browser_set_cookie` — domain field required (High · P1)

```
browser_navigate { url: "https://testtrack.org" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```

PASS if: `Cookie set: mcp_test=abc123`
FAIL if: `BiDi error: invalid argument - invalid argument`

Verify cookie appears in `browser_get_cookies`:
```
browser_get_cookies {}
```
PASS if: `mcp_test=abc123` visible; FAIL if: cookie missing or error

Cleanup:
```
browser_delete_cookies { name: "mcp_test" }
```

**Cross-site hardening:**

```
# Coffee Cart
browser_navigate { url: "https://coffee-cart.app/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# Swag Labs
browser_navigate { url: "https://www.saucedemo.com" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# AcademyBugs
browser_navigate { url: "https://academybugs.com/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# The Internet
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# Practice Software Testing
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# DemoQA
browser_navigate { url: "https://demoqa.com/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

```
# Practice Test Automation
browser_navigate { url: "https://practicetestautomation.com/practice-test-login/" }
browser_set_cookie { name: "mcp_test", value: "abc123" }
```
PASS if: cookie set; FAIL if: argument error

PASS (cross-site) if: cookie set successfully on all 7 sites
FAIL (cross-site) if: argument error on any site

---

### MB5 — `browser_get_attribute` — null/absent attribute causes MCP serialization error (High · P2)

When an attribute is absent, the tool returns a large `invalid_union` JSON error instead of an empty string or null.

```
browser_navigate { url: "https://testtrack.org/button-demo" }
browser_get_attribute { selector: "#disabled-button", attribute: "disabled" }
```
PASS if: returns `""`, `null`, `"true"`, or any non-error value
FAIL if: large `invalid_union` JSON error block

```
browser_get_attribute { selector: "#primary-button", attribute: "data-nonexistent" }
```
PASS if: returns `""` or `null`; FAIL if: `invalid_union` error

Workaround — use `browser_evaluate` instead:
```
browser_evaluate { expression: "document.querySelector('#disabled-button').hasAttribute('disabled').toString()" }
```
PASS (workaround) if: returns `"true"` or `"false"`

**Cross-site hardening** — test absent/boolean attributes across sites:

```
# Swag Labs — product images have alt; out-of-stock items have disabled add-to-cart
browser_navigate { url: "https://www.saucedemo.com" }
browser_fill { selector: "#user-name", text: "standard_user" }
browser_fill { selector: "#password", text: "secret_sauce" }
browser_click { selector: "#login-button" }
browser_get_attribute { selector: "img.inventory_item_img", attribute: "alt" }
```
PASS if: returns the alt text string; FAIL if: `invalid_union` error

```
# The Internet — basic auth link has no disabled attribute
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_get_attribute { selector: "li a", attribute: "disabled" }
```
PASS if: returns `""` or `null` (attribute absent); FAIL if: `invalid_union` error

```
# AcademyBugs — test for absent data attribute
browser_navigate { url: "https://academybugs.com/" }
browser_get_attribute { selector: "a", attribute: "data-nonexistent-attr" }
```
PASS if: returns `""` or `null`; FAIL if: `invalid_union` error

```
# Practice Test Automation — login page input has required attribute
browser_navigate { url: "https://practicetestautomation.com/practice-test-login/" }
browser_get_attribute { selector: "#username", attribute: "required" }
```
PASS if: returns `""`, `null`, or `"required"` (not an error); FAIL if: `invalid_union` error

```
# DemoQA text-box — placeholder attr present (should return); disabled attr absent
browser_navigate { url: "https://demoqa.com/text-box" }
browser_get_attribute { selector: "#userName", attribute: "placeholder" }
```
PASS if: returns `"Full Name"`; FAIL if: `invalid_union` error

```
browser_get_attribute { selector: "#userName", attribute: "disabled" }
```
PASS if: returns `""` or `null` (absent); FAIL if: `invalid_union` error

```
# Practice Software Testing — absent data attribute
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_get_attribute { selector: "a", attribute: "data-nonexistent" }
```
PASS if: returns `""` or `null`; FAIL if: `invalid_union` error

PASS (cross-site) if: no serialization errors on any site; absent attributes return empty or null; present string attributes return their value
FAIL (cross-site) if: `invalid_union` error on any absent/null attribute

---

### MB6 — `browser_evaluate` — empty string result causes MCP serialization error (High · P2)

Shares root cause with MB5. Specifically: `null` and `undefined` JS results serialize correctly, but empty string `""` does not.

```
browser_navigate { url: "https://testtrack.org" }
browser_evaluate { expression: "document.querySelector('#nonexistent')?.id" }
```
PASS if: returns `null` (null serializes fine)
FAIL if: `invalid_union` MCP serialization error

```
browser_evaluate { expression: "Array.from(document.querySelectorAll('[id]')).map(e => e.id).filter(id => id.includes('zzz_nonexistent')).join(', ')" }
```
PASS if: returns `""` (empty string); FAIL if: `invalid_union` error

Workaround — wrap result in `JSON.stringify` to guarantee string output:
```
browser_evaluate { expression: "JSON.stringify(document.querySelector('#nonexistent')?.id ?? null)" }
```
PASS (workaround) if: returns `"null"` (string, not null itself)

**Cross-site hardening:**

```
# Coffee Cart — querySelector miss on nonexistent element
browser_navigate { url: "https://coffee-cart.app/" }
browser_evaluate { expression: "document.querySelector('#nonexistent-element')?.textContent" }
```
PASS if: non-error; FAIL if: `invalid_union` error

```
# Swag Labs — optional chaining on missing property
browser_navigate { url: "https://www.saucedemo.com" }
browser_evaluate { expression: "document.querySelector('.nonexistent')?.dataset?.id" }
```
PASS if: non-error; FAIL if: `invalid_union` error

```
# The Internet — undefined variable reference (should return undefined gracefully)
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_evaluate { expression: "window.__nonExistentVar" }
```
PASS if: non-error; FAIL if: `invalid_union` error

```
# AcademyBugs — filter that returns empty array joined to empty string
browser_navigate { url: "https://academybugs.com/" }
browser_evaluate { expression: "Array.from(document.querySelectorAll('button')).filter(b => b.id === 'zzznonexistent').map(b => b.id).join('')" }
```
PASS if: returns `""`; FAIL if: `invalid_union` error

```
# DemoQA — empty string from filter+join
browser_navigate { url: "https://demoqa.com/" }
browser_evaluate { expression: "Array.from(document.querySelectorAll('h5')).filter(h => h.textContent === 'zzz_nonexistent').map(h => h.id).join('')" }
```
PASS if: returns `""`; FAIL if: `invalid_union` error

```
# Practice Software Testing — empty string from filter+join
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_evaluate { expression: "Array.from(document.querySelectorAll('a')).filter(a => a.href === 'https://zzz.nonexistent.example/').map(a => a.textContent).join('')" }
```
PASS if: returns `""`; FAIL if: `invalid_union` error

PASS (cross-site) if: no `invalid_union` errors; note null results are fine, only empty string `""` triggers the bug
FAIL (cross-site) if: `invalid_union` error on any empty string result

---

### MB7 — `browser_fill` — fails on `<textarea>` elements (High · P2)

```
browser_set_content { html: "<html><body><textarea id='ta'></textarea></body></html>" }
browser_fill { selector: "#ta", text: "hello textarea" }
browser_get_value { selector: "#ta" }
```
PASS if: fill succeeds and `get_value` returns `"hello textarea"`
FAIL if: `failed to fill:` error

Workaround — use `browser_type` after `browser_click`:
```
browser_set_content { html: "<html><body><textarea id='ta'></textarea></body></html>" }
browser_click { selector: "#ta" }
browser_type { selector: "#ta", text: "workaround text" }
browser_get_value { selector: "#ta" }
```
PASS (workaround) if: value is `"workaround text"`

**Cross-site hardening** — real-world textareas:

```
# Evil Tester basic HTML form — textarea element
browser_navigate { url: "https://testpages.eviltester.com/styled/basic-html-form-test.html" }
browser_find { selector: "textarea" }
browser_fill { selector: "@e1", text: "evil tester textarea test" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error

```
# Automation In Testing contact form — #description textarea
browser_navigate { url: "https://automationintesting.online/#/" }
browser_find { selector: "textarea#description" }
browser_fill { selector: "textarea#description", text: "test message from vibium mcp" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error
Note: workaround for this site is `browser_evaluate { expression: "const el=document.querySelector('#description'); el.value='text'; el.dispatchEvent(new Event('input',{bubbles:true}))" }`

```
# Testtrack text input demo — textarea with word count
browser_navigate { url: "https://testtrack.org/text-input-demo" }
browser_fill { selector: "textarea", text: "textarea fill test on testtrack" }
browser_find { text: "WORD COUNT:" }
```
PASS if: fill succeeds and word count updates; FAIL if: `failed to fill:` error

```
# QA Training Simulator — multi-line textarea input
browser_navigate { url: "https://bugeater.web.app/app/challenge/learn/adder" }
browser_find { selector: "textarea" }
```
Skip if no textarea found on this page. If found:
```
browser_fill { selector: "textarea", text: "test" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error; SKIP if: no textarea present

```
# Practice Automation — form page textarea
browser_navigate { url: "https://practice-automation.com/form-fields/" }
browser_find { selector: "textarea" }
browser_fill { selector: "textarea", text: "practice automation textarea" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error

```
# DemoQA text-box — textarea for current address
browser_navigate { url: "https://demoqa.com/text-box" }
browser_fill { selector: "#currentAddress", text: "demoqa textarea test" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error

```
# Automation Testing Practice — textarea on blogspot form
browser_navigate { url: "https://testautomationpractice.blogspot.com/" }
browser_fill { selector: "textarea", text: "blogspot textarea test" }
```
PASS if: fill succeeds; FAIL if: `failed to fill:` error

PASS (cross-site) if: `browser_fill` works on all textareas found across sites
FAIL (cross-site) if: `failed to fill:` error on any textarea; workaround (`browser_type`) required

---

### MB8 — `browser_screenshot` with `annotate: true` — script exception (Medium · P3)

```
browser_navigate { url: "https://testtrack.org" }
browser_screenshot { annotate: true }
```
PASS if: screenshot returned with numbered labels on interactive elements
FAIL if: `failed to annotate: script exception:` — annotation script crashes

Verify unannotated screenshot still works on same page:
```
browser_screenshot { annotate: false }
```
PASS if: screenshot returned normally

**Cross-site hardening** — annotation across different frontend stacks:

```
# Coffee Cart — Vue SPA with product cards and cart nav
browser_navigate { url: "https://coffee-cart.app/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot with numbered elements; FAIL if: script exception

```
# Swag Labs — React app after login
browser_navigate { url: "https://www.saucedemo.com" }
browser_fill { selector: "#user-name", text: "standard_user" }
browser_fill { selector: "#password", text: "secret_sauce" }
browser_click { selector: "#login-button" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

```
# AcademyBugs — vanilla JS with many interactive elements
browser_navigate { url: "https://academybugs.com/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

```
# Practice Test Automation — simple login page
browser_navigate { url: "https://practicetestautomation.com/practice-test-login/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

```
# The Internet — link-heavy page
browser_navigate { url: "https://the-internet.herokuapp.com/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

```
# DemoQA — component library with many interactive elements
browser_navigate { url: "https://demoqa.com/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

```
# Practice Software Testing — e-commerce with product listing
browser_navigate { url: "https://practicesoftwaretesting.com/" }
browser_screenshot { annotate: true }
```
PASS if: annotated screenshot; FAIL if: script exception

PASS (cross-site) if: annotated screenshots produced on all 7 sites
FAIL (cross-site) if: script exception on any site

---

### MB9 — `browser_get_text` — invalid_union error on blank pages (Medium · P3)

```
# Navigate to a blank page (React SPA routing miss)
browser_navigate { url: "https://www.cnarios.com/concepts/iframe" }
browser_wait_for_load {}
browser_get_text {}
```
PASS if: returns `""` (empty string)
FAIL if: `invalid_union` schema error — "Invalid input: expected string, received undefined"

Verify that `browser_get_text` works normally on a page with content:
```
browser_navigate { url: "https://www.cnarios.com/" }
browser_wait_for_load {}
browser_get_text {}
```
PASS if: page text returned normally

**Cross-site hardening** — blank page scenarios on different stacks:

```
# Cnarios /concepts/multi-window — another React routing miss
browser_navigate { url: "https://www.cnarios.com/concepts/multi-window" }
browser_wait_for_load {}
browser_get_text {}
```
PASS if: returns `""`; FAIL if: `invalid_union` error

**Workaround verification** — confirm `browser_evaluate` with coercion avoids the bug:
```
browser_navigate { url: "https://www.cnarios.com/concepts/iframe" }
browser_wait_for_load {}
browser_evaluate { expression: "document.body.innerText || null" }
```
PASS if: returns `null` (no schema error — null serializes correctly unlike `""`)

PASS (cross-site) if: `browser_get_text` returns `""` on all blank pages tested
FAIL (cross-site) if: `invalid_union` error on any blank page; workaround required

---

## Cleanup

```
browser_navigate { url: "https://testtrack.org" }
browser_delete_cookies {}
```

---

## Final Report

Print a summary table with actual results filled in:

```
╔═══════════════════════════════════════════════════════════════════════╗
║               vibium MCP REGRESSION TEST RESULTS                      ║
╠══════╦══════════╦══════════╦════════════════════════════════════════╣
║ Bug  ║ Severity ║ Priority ║ Result                                 ║
╠══════╬══════════╬══════════╬════════════════════════════════════════╣
║ MB1  ║ Critical ║ P1       ║ PASS / FAIL / SKIP                     ║
║ MB2  ║ Critical ║ P1       ║ PASS / FAIL / SKIP                     ║
║ MB3  ║ Critical ║ P1       ║ PASS / FAIL / SKIP                     ║
║ MB4  ║ High     ║ P1       ║ PASS / FAIL / SKIP                     ║
║ MB5  ║ High     ║ P2       ║ PASS / FAIL / SKIP                     ║
║ MB6  ║ High     ║ P2       ║ PASS / FAIL / SKIP                     ║
║ MB7  ║ High     ║ P2       ║ PASS / FAIL / SKIP                     ║
║ MB8  ║ Medium   ║ P3       ║ PASS / FAIL / SKIP                     ║
║ MB9  ║ Medium   ║ P3       ║ PASS / FAIL / SKIP                     ║
╠══════╩══════════╩══════════╩════════════════════════════════════════╣
║  X PASS   Y FAIL   Z SKIP   (9 total)                                 ║
╚═══════════════════════════════════════════════════════════════════════╝
```

Cross-site hardening summary — for each FAIL, include:
- Which site(s) triggered it
- The exact error output observed
- Whether the symptom matches or differs from the original bug report
- Whether the documented workaround still resolves it
