# browser_get_text — invalid_union error when page has no text content (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**

## Summary

`browser_get_text` (no selector — full page) throws an `invalid_union` MCP serialization error when the page has no text content. On a blank page, `document.body.innerText` returns `""`. The MCP serializer cannot produce a valid content block from an empty string — same root cause as [MB6](./MB6.md) (`browser_evaluate` empty string result) — but unlike MB6, the user cannot add a fallback because the internal expression is not exposed.

## Repro

```
# Navigate to a page with no rendered text (e.g. React SPA with broken routing)
browser_navigate { url: "https://www.cnarios.com/concepts/iframe" }
browser_wait_for_load {}
browser_get_text {}
```

**Error:**
```json
[{"code":"invalid_union","errors":[[{"expected":"string","code":"invalid_type","path":["text"],"message":"Invalid input: expected string, received undefined"}], ...]}]
```

## Expected

Returns `""` (empty string) when page has no text content.

## Actual

Returns `invalid_union` serialization error. Page is left in a navigated state; no text value returned.

## Scope

Confirmed on blank pages (React routing misses, intentionally empty pages):

| Site | URL | Page state | Result |
|------|-----|------------|--------|
| cnarios.com | /concepts/iframe | React routing miss — blank page | `invalid_union` |
| cnarios.com | /concepts/multi-window | React routing miss — blank page | `invalid_union` (expected) |
| Any site | Any page with content | Normal rendered text | ✓ returns text |

## Root cause

Same as MB6: Go MCP content union serializer cannot produce a `text` content block from an empty string — it treats `""` as `undefined`, failing schema validation. `browser_get_text` internally calls the equivalent of `document.body.innerText`, which returns `""` on a blank page. Unlike `browser_evaluate`, the caller cannot modify the expression to add a non-empty fallback.

## Workaround

Use `browser_evaluate` with string coercion instead:
```
browser_evaluate { expression: "document.body.innerText || ''" }
```
This returns `null` (not `""`) when empty, which the serializer handles correctly. Or use a sentinel:
```
browser_evaluate { expression: "document.body.innerText || '__empty__'" }
```

## Relationship to MB6

MB6 covers `browser_evaluate` returning `""`. MB9 is the same root cause manifesting in `browser_get_text`, where the workaround is not available in-band. Both should be fixed by the same underlying serializer fix (treat `""` as a valid empty string, not `undefined`).

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB9
