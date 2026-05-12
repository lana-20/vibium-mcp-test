# browser_set_cookie — fails without explicit domain field (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#152

## Summary

`browser_set_cookie` always fails with a BiDi `invalid argument` error when called without an explicit `domain` parameter. The tool schema marks `domain` as optional, but the underlying BiDi call requires it. Every cookie set attempt fails regardless of the current page URL.

## Repro

```
browser_navigate { url: "https://testtrack.org" }
browser_set_cookie { name: "test_cookie", value: "abc123" }
```

**Error:**
```
failed to set cookie: BiDi error: invalid argument - invalid argument
```

## Expected

Cookie set using the current page's domain when `domain` is omitted, consistent with browser devtools and Playwright behavior.

## Actual

Fails with `invalid argument` on every site. The tool never falls back to the current page's domain.

## Scope

Confirmed on 7 sites:

| Site | Result |
|------|--------|
| testtrack.org | invalid argument |
| coffee-cart.app | invalid argument |
| saucedemo.com | invalid argument |
| academybugs.com | invalid argument |
| the-internet.herokuapp.com | invalid argument |
| practicesoftwaretesting.com | invalid argument |
| demoqa.com | invalid argument |

## Workaround

Pass `domain` explicitly matching the current page:
```
browser_set_cookie { name: "test_cookie", value: "abc123", domain: "testtrack.org" }
```

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB4
