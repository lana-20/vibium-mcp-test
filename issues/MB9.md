# browser_get_text — invalid_union error when page/element has no text content (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**

## Summary

`browser_get_text` throws an `invalid_union` MCP serialization error whenever the target's text content is empty (`""`). This affects both the full-page form (no selector) and the element form (with selector). `document.body.innerText` returns `""` on blank pages, whitespace-only pages, pages with only hidden elements (`display:none` / `visibility:hidden`), and empty DOM elements. The MCP serializer cannot produce a valid content block from `""` — same root cause as [MB6](./MB6.md) — but unlike MB6, the caller cannot fix it by modifying the expression, because `browser_get_text` does not expose its internal script.

Note: a nonexistent selector gives a different, correct error: `"failed to get text: element not found"`. MB9 is only triggered when the element exists but its text content is empty.

## Repro

**Full-page form:**
```
browser_navigate { url: "https://www.cnarios.com/concepts/iframe" }
browser_wait_for_load {}
browser_get_text {}
```

**Element form:**
```
browser_set_content { html: "<html><body><div id='e'></div></body></html>" }
browser_get_text { selector: "#e" }
```

**Error (identical in both forms):**
```json
[{"code":"invalid_union","errors":[[{"expected":"string","code":"invalid_type","path":["text"],"message":"Invalid input: expected string, received undefined"}], ...]}]
```

## Expected

Returns `""` (empty string) when text content is empty.

## Actual

Returns `invalid_union` serialization error. No text value returned.

## Scope

Hardened across 9 scenarios (2026-05-18):

| # | Scenario | Trigger | Result |
|---|----------|---------|--------|
| S1 | React SPA routing miss | `cnarios.com/concepts/iframe` (blank `#root`) | FAIL `invalid_union` |
| S2 | `about:blank` | `browser_navigate { url: "about:blank" }` | FAIL `invalid_union` |
| S3 | Empty body via `set_content` | `<html><body></body></html>` | FAIL `invalid_union` |
| S4 | Whitespace-only body | `<html><body>   </body></html>` | FAIL `invalid_union` |
| S5 | Empty element (selector form) | `browser_get_text { selector: "#empty-div" }` | FAIL `invalid_union` |
| S6 | Non-empty element (selector form) | `browser_get_text { selector: "#content" }` where content = "hello" | PASS ✓ |
| S7 | Nonexistent selector | `browser_get_text { selector: "#does-not-exist" }` | `element not found` (different error, correct) |
| S8 | Dynamically emptied real page | coffee-cart.app with `body.innerHTML = ''` via eval | FAIL `invalid_union` |
| S9 | Hidden-only content | `<div style="display:none">text</div>` — `innerText` skips hidden elements | FAIL `invalid_union` |

S9 is particularly insidious: the page has DOM content, but all of it is hidden. `innerText` (which respects CSS visibility) returns `""`, triggering the bug even though `innerHTML` would be non-empty.

Additional check — `cnarios.com/concepts/multi-window`: now renders a 404 page with visible text (React routing partially fixed on this route) — `browser_get_text` returns content normally. Not a valid repro for MB9 anymore.

## Root cause

Same as MB6: Go MCP content union serializer cannot produce a `text` content block from `""` — it treats the empty string as `undefined`, failing schema validation. `browser_get_text` internally reads `innerText` (or equivalent), which returns `""` for empty/hidden content. Unlike `browser_evaluate`, the caller cannot add a fallback expression.

## Workaround

Use `browser_evaluate` with `|| null` fallback — `null` serializes correctly, `""` does not:

```
# Full-page replacement
browser_evaluate { expression: "document.body.innerText || null" }

# Element replacement
browser_evaluate { expression: "document.querySelector('#e').innerText || null" }
```

**Do not use `|| ''`** — that would return an empty string and trigger MB6.

Workaround confirmed working across S1, S2, S4 (hidden-only), and S5 (empty element) — all return `null` with no schema error.

## Relationship to MB6

MB6: `browser_evaluate` returning `""` → `invalid_union`
MB9: `browser_get_text` reading empty text → `invalid_union`

Same serializer bug. MB9 is less fixable by the caller. Both are resolved by the same underlying fix: treat `""` as a valid empty-string content block rather than coercing it to `undefined`.

## Re-verification history

| Date | Version | Result | Notes |
|------|---------|--------|-------|
| 2026-05-18 | v26.3.18 | FAIL (S1–S5, S8–S9) | Initial hardening across 9 scenarios |
| 2026-05-19 | v26.3.18 | FAIL (S1–S7 all) | Re-run via `/vibium-mcp-test`; all 7 SKILL.md scenarios confirmed; error string identical; workaround (`|| null`) confirmed still working |

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB9
