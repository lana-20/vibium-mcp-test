# browser_get_text — invalid_union error when page/element has no text content (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**

## Summary

`browser_get_text` throws an `invalid_union` MCP serialization error whenever the target's text content is empty (`""`). This affects both the full-page form (no selector) and the element form (with selector). `document.body.innerText` returns `""` on blank pages, whitespace-only pages, pages with only hidden elements (`display:none` / `visibility:hidden`), and empty DOM elements. The MCP serializer cannot produce a valid content block from `""` — same root cause as #154 — but unlike `browser_evaluate`, the caller cannot fix it by modifying the expression, because `browser_get_text` does not expose its internal script.

Note: a nonexistent selector gives a different, correct error: `"failed to get text: element not found"`. This bug is only triggered when the element exists but its text content is empty.

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
[{"code":"invalid_union","errors":[[{"expected":"string","code":"invalid_type","path":["text"],"message":"Invalid input: expected string, received undefined"}],[{"code":"invalid_value","values":["image"],"path":["type"],"message":"Invalid input: expected \"image\""},{"expected":"string","code":"invalid_type","path":["data"],"message":"Invalid input: expected string, received undefined"},{"expected":"string","code":"invalid_type","path":["mimeType"],"message":"Invalid input: expected string, received undefined"}],[{"code":"invalid_value","values":["audio"],"path":["type"],"message":"Invalid input: expected \"audio\""},{"expected":"string","code":"invalid_type","path":["data"],"message":"Invalid input: expected string, received undefined"},{"expected":"string","code":"invalid_type","path":["mimeType"],"message":"Invalid input: expected string, received undefined"}],[{"expected":"string","code":"invalid_type","path":["name"],"message":"Invalid input: expected string, received undefined"},{"expected":"string","code":"invalid_type","path":["uri"],"message":"Invalid input: expected string, received undefined"},{"code":"invalid_value","values":["resource_link"],"path":["type"],"message":"Invalid input: expected \"resource_link\""}],[{"code":"invalid_value","values":["resource"],"path":["type"],"message":"Invalid input: expected \"resource\""},{"code":"invalid_union","errors":[[{"expected":"object","code":"invalid_type","path":[],"message":"Invalid input: expected object, received undefined"}],[{"expected":"object","code":"invalid_type","path":[],"message":"Invalid input: expected object, received undefined"}]],"path":["resource"],"message":"Invalid input"}]],"path":["content",0],"message":"Invalid input"}]
```

## Expected

Returns `""` (empty string) when text content is empty.

## Actual

Returns `invalid_union` serialization error. No text value returned.

## Scope

Verified across 7 scenarios:

| # | Scenario | Trigger | Result |
|---|----------|---------|--------|
| S1 | React SPA routing miss | `cnarios.com/concepts/iframe` (blank `#root`) | FAIL `invalid_union` |
| S2 | `about:blank` | `browser_navigate { url: "about:blank" }` | FAIL `invalid_union` |
| S3 | Empty body via `set_content` | `<html><body></body></html>` | FAIL `invalid_union` |
| S4 | Whitespace-only body | `<html><body>   </body></html>` | FAIL `invalid_union` |
| S5 | Empty element (selector form) | `browser_get_text { selector: "#e" }` on `<div id='e'></div>` | FAIL `invalid_union` |
| S6 | Hidden-only content | `<div style="display:none">text</div>` — `innerText` skips hidden elements | FAIL `invalid_union` |
| S7 | Dynamically emptied real page | coffee-cart.app with `body.innerHTML = ''` via eval | FAIL `invalid_union` |

S6 is particularly insidious: the page has DOM content, but all of it is hidden. `innerText` (which respects CSS visibility) returns `""`, triggering the bug even though `innerHTML` is non-empty.

Control — non-empty element returns correctly:
```
browser_set_content { html: "<html><body><div id='e'></div><div id='c'>hello</div></body></html>" }
browser_get_text { selector: "#c" }
```
Returns `"hello"` — selector form works when content is non-empty.

## Root cause

Same as #154: Go MCP content union serializer cannot produce a `text` content block from `""` — it treats the empty string as `undefined`, failing schema validation. `browser_get_text` internally reads `innerText` (or equivalent), which returns `""` for empty/hidden content. Unlike `browser_evaluate`, the caller cannot add a fallback expression.

## Workaround

Use `browser_evaluate` with `|| null` fallback — `null` serializes correctly, `""` does not:

```
# Full-page replacement
browser_evaluate { expression: "document.body.innerText || null" }

# Element replacement
browser_evaluate { expression: "document.querySelector('#e').innerText || null" }
```

**Do not use `|| ''`** — that returns an empty string and triggers #154.

Workaround confirmed working across all 7 scenarios — all return `null` with no schema error.

## Relationship to #154

\#154: `browser_evaluate` returning `""` → `invalid_union`
This issue: `browser_get_text` reading empty text → `invalid_union`

Same serializer bug. This issue is less fixable by the caller. Both are resolved by the same underlying fix: treat `""` as a valid empty-string content block rather than coercing it to `undefined`.
