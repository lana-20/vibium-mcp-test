# browser_count — Go type mismatch crash on every selector (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#149

## Summary

`browser_count` always fails with a Go type unmarshal error. The BiDi script result returns a JavaScript number, but the Go MCP layer expects a string field — causing a parse crash on every invocation regardless of page or selector.

## Repro

```
browser_navigate { url: "https://testtrack.org" }
browser_count { selector: "a" }
```

**Error:**
```
failed to count: failed to parse script result: json: cannot unmarshal number into Go struct field .result.result.value of type string
```

## Expected

Returns an integer count, e.g. `42`.

## Actual

Crashes with unmarshal error on every site and selector tested.

## Scope

Confirmed on 7 sites across Vue SPA, React, vanilla JS, Angular-like, and e-commerce stacks:

| Site | Selector | Result |
|------|----------|--------|
| testtrack.org | `a` | unmarshal error |
| coffee-cart.app | `button` | unmarshal error |
| saucedemo.com | `button` | unmarshal error |
| academybugs.com | `a` | unmarshal error |
| the-internet.herokuapp.com | `li a` | unmarshal error |
| automationexercise.com | `button` | unmarshal error |
| practicesoftwaretesting.com | `a` | unmarshal error |

## Root cause

`browser_count` evaluates a script that returns a JS number. The Go struct field receiving `.result.result.value` is typed as `string`. `json.Unmarshal` cannot coerce a JSON number into a Go string — it errors instead.

## Workaround

Use `browser_evaluate` with `.toString()` to force a string return:
```
browser_evaluate { expression: "document.querySelectorAll('a').length.toString()" }
```

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB1
