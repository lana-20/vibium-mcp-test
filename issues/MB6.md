# browser_evaluate — invalid_union error when expression returns empty string "" (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#154

## Summary

`browser_evaluate` returns an `invalid_union` MCP serialization error when the JS expression evaluates to an empty string `""`. Same root cause as [MB5](https://github.com/VibiumDev/vibium/issues/153) — the Go MCP serializer cannot produce a valid content union from an empty string. Importantly, `null` and `undefined` results serialize correctly; only `""` triggers the bug.

## Repro

```
browser_navigate { url: "https://testtrack.org" }
browser_evaluate { expression: "Array.from(document.querySelectorAll('[id]')).map(e => e.id).filter(id => id.includes('zzz_nonexistent')).join(', ')" }
```

The filter matches nothing; `.join()` on an empty array returns `""`.

**Error (truncated):**
```json
[{"code":"invalid_union","errors":[[{"expected":"string","code":"invalid_type","path":["text"],"message":"Invalid input: expected string, received undefined"}], ...]}]
```

## Expected

Returns `""` (empty string).

## Actual

Returns `invalid_union` serialization error.

## Scope

Confirmed on 4 of 6 tested sites — specifically when expression returns `""`. `null` results handled correctly on all sites.

| Site | Expression result | Result |
|------|-------------------|--------|
| testtrack.org | `null` (optional chain miss) | ✓ returns null |
| testtrack.org | `""` (empty join) | `invalid_union` |
| coffee-cart.app | `null` | ✓ returns null |
| saucedemo.com | `null` | ✓ returns null |
| the-internet.herokuapp.com | `null` (undefined var) | ✓ returns null |
| academybugs.com | `""` (empty join) | `invalid_union` |
| demoqa.com | `""` (empty join) | `invalid_union` |
| practicesoftwaretesting.com | `""` (empty join) | `invalid_union` |

## Root cause

Same as [MB5](https://github.com/VibiumDev/vibium/issues/153): Go MCP content union serializer cannot produce a `text` content block from an empty string — it treats it as `undefined` rather than an empty string value.

## Workaround

Ensure the expression never returns `""` — use a fallback:
```
browser_evaluate { expression: "JSON.stringify(someExpression ?? null)" }
browser_evaluate { expression: "someFilteredArray.join('') || 'empty'" }
```

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB6
