# browser_click — deadlocks BiDi socket when native JS dialog fires (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#151

## Summary

`browser_click` on any element that triggers a native JS dialog (`alert`, `confirm`, `prompt`) deadlocks the BiDi socket. The call never returns. The browser session must be manually quit and restarted to recover.

## Repro

```
browser_navigate { url: "https://the-internet.herokuapp.com/javascript_alerts" }
browser_find { role: "button", text: "Click for JS Alert" }
browser_click { selector: "@e1" }
```

**Result:** `browser_click` hangs indefinitely. No error is returned. Browser session is deadlocked.

**Recovery:** manually quit the browser process, then call `browser_stop {}` and `browser_start {}`.

## Expected

`browser_click` returns after the click. A subsequent `browser_dialog_accept {}` or `browser_dialog_dismiss {}` handles the open dialog.

## Actual

`browser_click` blocks the BiDi socket waiting for the page to settle. The page cannot settle while a native dialog is open. Circular wait — never resolves.

## Scope

Confirmed on all 4 sites tested with direct click:

| Site | Button | Dialog type | Result |
|------|--------|-------------|--------|
| the-internet.herokuapp.com/javascript_alerts | "Click for JS Alert" | `alert` | deadlock |
| testpages.eviltester.com/styled/alerts/alert-test.html | "Show alert box" | `alert` | deadlock |
| testautomationpractice.blogspot.com | "Simple Alert" | `alert` | deadlock |
| testtrack.org/alert-demo | `#temporal-alert-btn` | `alert` | deadlock |

This is a BiDi-level bug — all sites that trigger native dialogs via `browser_click` deadlock. Not site-specific.

## Workarounds

Two workarounds confirmed working on all 6 tested sites:

**1. setTimeout pattern** — schedule dialog asynchronously so `browser_evaluate` returns before it fires:
```
browser_evaluate { expression: "setTimeout(() => alert('test'), 300)" }
browser_dialog_accept {}
```

**2. Pre-stub pattern** — override `window.alert/confirm/prompt` before clicking so no native dialog fires:
```
browser_evaluate { expression: "window.alert = () => {}; window.confirm = () => true; window.prompt = () => 'value'" }
browser_click { selector: "#trigger-button" }
```

Note: there is no MCP tool to pre-register a dialog handler. `browser_dialog_accept`/`browser_dialog_dismiss` only handle an already-open dialog, not future ones.

## See also

- #146 — same deadlock in the Python client (`capture.dialog` + `page.evaluate("alert(...)")`)

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB3
