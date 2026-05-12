# browser_storage_state — cookie object parse error on all sites (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#150

## Summary

`browser_storage_state` always fails with a Go type unmarshal error. The BiDi `storage.getCookies` result returns cookie objects with structured fields (e.g. `SameSite` as an object), but the Go struct expects the `value` field to be a plain string — causing a parse crash on every site.

## Repro

```
browser_navigate { url: "https://testtrack.org" }
browser_storage_state {}
```

**Error:**
```
failed to get cookies: failed to parse storage.getCookies result: json: cannot unmarshal object into Go struct field Cookie.cookies.value of type string
```

## Expected

Returns JSON with `cookies`, `localStorage`, and `sessionStorage` fields.

## Actual

Crashes with unmarshal error on every site tested. `browser_restore_storage` is also blocked — no valid state file can be produced until this is fixed.

## Scope

Confirmed on 6 sites including cookie-rich and post-login sessions:

| Site | Condition | Result |
|------|-----------|--------|
| testtrack.org | baseline | unmarshal error |
| saucedemo.com | after login (session cookie set) | unmarshal error |
| coffee-cart.app | Vue SPA | unmarshal error |
| academybugs.com | analytics/banner cookies | unmarshal error |
| the-internet.herokuapp.com | Heroku app | unmarshal error |
| practicesoftwaretesting.com | e-commerce session | unmarshal error |

## Root cause

The BiDi `storage.getCookies` result contains cookie objects where some fields (e.g. `SameSite`) are themselves objects rather than strings. The Go `Cookie` struct has a `value` field typed as `string`, which cannot accept a JSON object.

## Workaround

None available from the call site. The deserialization happens inside the MCP layer before results are returned.

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB2
