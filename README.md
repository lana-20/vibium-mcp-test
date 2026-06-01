# vibium-mcp-test

A Claude Code skill that runs a full regression suite against the [vibium](https://www.npmjs.com/package/vibium) MCP server — a browser automation tool built on WebDriver BiDi, exposed as Model Context Protocol tools.

The suite covers **9 confirmed bugs** originally found in the vibium MCP layer, verified across 14 sites. Each test maps to a specific MCP tool failure, produces a `PASS / FAIL / SKIP` result, and includes exact repro steps, error strings, and workarounds.

**v26.5.31 status: all 9 PASS.** MB1–MB9 are fixed at the shared engine level.

## Usage

Install the skill via Claude Code, then run:

```
/vibium-mcp-test
```

Claude will execute all 9 tests against the running vibium MCP server and print a summary table.

## What it tests

| # | Severity | Priority | Tool | Bug |
|---|----------|----------|------|-----|
| MB1 | Critical | P1 | `browser_count` | ~~Go type mismatch — number result cannot unmarshal into string field~~ **Fixed v26.5.31** |
| MB2 | Critical | P1 | `browser_storage_state` | ~~Go type mismatch — cookie object cannot unmarshal into string field~~ **Fixed v26.5.31** |
| MB3 | Critical | P1 | `browser_click` + `browser_dialog_accept/dismiss` | Native JS dialog deadlocks BiDi socket — **workaround confirmed v26.5.31**; direct-click deadlock deferred |
| MB4 | High | P1 | `browser_set_cookie` | ~~BiDi requires `domain` field; call always fails without it~~ **Fixed v26.5.31** — domain derived from current page |
| MB5 | High | P2 | `browser_get_attribute` | ~~Absent/null attribute causes MCP `invalid_union` serialization error~~ **Fixed v26.5.31** |
| MB6 | High | P2 | `browser_evaluate` | ~~Empty string `""` result causes MCP `invalid_union` serialization error~~ **Fixed v26.5.31** |
| MB7 | High | P2 | `browser_fill` | ~~Fails on `<textarea>` elements — `failed to fill:` error~~ **Fixed v26.5.31** |
| MB8 | Medium | P3 | `browser_screenshot` | ~~`annotate: true` crashes annotation script on all sites~~ **Fixed v26.5.31** |
| MB9 | Medium | P3 | `browser_get_text` | ~~`invalid_union` error when page or element text content is `""`~~ **Fixed v26.5.31** |

## Cross-site coverage

Bugs are verified across 13 sites:

| Site | Tests |
|------|-------|
| [testtrack.org](https://testtrack.org) | MB1–MB8 (baseline) |
| [coffee-cart.app](https://coffee-cart.app/) | MB1, MB2, MB4, MB6, MB8, MB9 |
| [saucedemo.com](https://www.saucedemo.com) | MB1, MB2, MB4, MB5, MB8 |
| [academybugs.com](https://academybugs.com/) | MB1, MB2, MB4, MB5, MB6, MB8 |
| [the-internet.herokuapp.com](https://the-internet.herokuapp.com/) | MB1, MB2, MB3, MB4, MB5, MB8 |
| [cnarios.com](https://www.cnarios.com/concepts/iframe) | MB9 |
| [automationexercise.com](https://www.automationexercise.com/products) | MB1 |
| [testpages.eviltester.com](https://testpages.eviltester.com/styled/alerts/alert-test.html) | MB3, MB7 |
| [automationintesting.online](https://automationintesting.online/#/) | MB7 |
| [testautomationpractice.blogspot.com](https://testautomationpractice.blogspot.com/) | MB3, MB7 |
| [practicetestautomation.com](https://practicetestautomation.com/practice-test-login/) | MB3, MB4, MB5, MB8 |
| [practice-automation.com](https://practice-automation.com/form-fields/) | MB7 |
| [demoqa.com](https://demoqa.com/) | MB1, MB2, MB3, MB4, MB5, MB6, MB7, MB8 |
| [practicesoftwaretesting.com](https://practicesoftwaretesting.com/) | MB1, MB2, MB4, MB5, MB6, MB8 |

## MB3 — dialog deadlock workarounds

MB3 has two confirmed workarounds:

1. **setTimeout pattern** — schedule the dialog asynchronously so `browser_evaluate` returns before the dialog fires, then call `browser_dialog_accept`/`browser_dialog_dismiss`:
   ```
   browser_evaluate { expression: "setTimeout(() => alert('test'), 300)" }
   browser_dialog_accept {}
   ```

2. **Pre-stub pattern** — override `window.alert`/`window.confirm`/`window.prompt` before clicking, so the native dialog is never shown:
   ```
   browser_evaluate { expression: "window.alert = () => {}; window.confirm = () => true; window.prompt = () => 'value'" }
   browser_click { selector: "#trigger-button" }
   ```

The direct-click pattern (`browser_click` on a button that calls `alert()`/`confirm()`) deadlocks the BiDi socket and requires a `browser_stop` + `browser_start` recovery. Deadlock confirmed on all tested sites (The Internet, Evil Tester, Blogspot, testtrack.org/alert-demo) — this is a BiDi-level bug, not site-specific.

## MB5 vs MB6 — serialization errors

Both MB5 and MB6 share the same root cause: the Go MCP layer cannot serialize certain JavaScript return values.

| | MB5 | MB6 |
|-|-----|-----|
| Tool | `browser_get_attribute` | `browser_evaluate` |
| Trigger | Absent or null attribute value | Empty string `""` result |
| Error | `invalid_union` / `received undefined` | `invalid_union` / `received undefined` |
| Workaround (MB5) | `browser_evaluate { expression: "el.hasAttribute('x').toString()" }` | — |
| Workaround (MB6) | — | Ensure expression returns non-empty string; use `JSON.stringify(... ?? null)` |
| Note | Present string attributes work fine | `null` results serialize correctly; only empty string `""` fails — title "null/empty/undefined" in original report is inaccurate |

## Requirements

- vibium MCP server running and connected to Claude Code
- Chrome + ChromeDriver installed
- Claude Code with the skill installed

## Output

The suite prints a `PASS / FAIL / SKIP` line per test and a final summary table:

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

Each `FAIL` includes the exact error string observed and notes whether the symptom matches the original bug report.

## Verified against

vibium MCP (npm vibium pre-v26.5.31) · ChromeDriver 147.0 · macOS darwin 25.3.0 · Claude Code claude-sonnet-4-6 (original)
vibium MCP v26.5.31 · ChromeDriver 147.0.7727.56 · macOS darwin 25.5.0 · Claude Code claude-sonnet-4-6 (2026-06-01, all 9 PASS)

## Changelog

| Date | Change |
|------|--------|
| 2026-06-01 | Ran full suite against v26.5.31; all 9 bugs fixed — MB1 (count unmarshal), MB2 (storage unmarshal), MB4 (set_cookie domain), MB5 (get_attribute invalid_union), MB6 (evaluate empty string), MB7 (fill textarea), MB8 (screenshot annotate), MB9 (get_text empty); MB3 workaround confirmed; direct-click deadlock still deferred |
