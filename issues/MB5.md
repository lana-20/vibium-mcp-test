# browser_get_attribute — invalid_union error when attribute is absent or null (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#153

## Summary

`browser_get_attribute` returns a large `invalid_union` MCP serialization error when the requested attribute is absent from the element or its value is null/undefined. Present string attributes return correctly. The bug is in the Go MCP response serialization layer — it cannot handle a null/undefined JS result.

## Repro

```
browser_navigate { url: "https://testtrack.org/button-demo" }
browser_get_attribute { selector: "#disabled-button", attribute: "disabled" }
```

**Error (truncated):**
```json
[{"code":"invalid_union","errors":[[{"expected":"string","code":"invalid_type","path":["text"],"message":"Invalid input: expected string, received undefined"}], ...]}]
```

Also fails for non-existent attributes:
```
browser_get_attribute { selector: "#primary-button", attribute: "data-nonexistent" }
```

## Expected

Returns `""`, `null`, or `"true"` for boolean attributes; returns `""` or `null` for absent attributes.

## Actual

Returns a large `invalid_union` JSON error block for any absent or null attribute value.

## Scope

Confirmed on 6 sites for absent/null attributes. Present string attributes work correctly on all sites.

| Site | Attribute | Present? | Result |
|------|-----------|----------|--------|
| testtrack.org | `disabled` (boolean) | yes | `invalid_union` |
| testtrack.org | `data-nonexistent` | no | `invalid_union` |
| saucedemo.com | `alt` (present string) | yes | ✓ returns value |
| saucedemo.com | `data-nonexistent` | no | `invalid_union` |
| the-internet.herokuapp.com | `disabled` on `li a` | no | `invalid_union` |
| academybugs.com | `data-nonexistent-attr` | no | `invalid_union` |
| practicetestautomation.com | `required` | absent/empty | `invalid_union` |
| demoqa.com | `placeholder` (present) | yes | ✓ returns value |
| demoqa.com | `disabled` | no | `invalid_union` |
| practicesoftwaretesting.com | `data-nonexistent` | no | `invalid_union` |

## Root cause

The Go MCP layer serializes the tool result as a content union type (`text`, `image`, `audio`, `resource`). When the JS attribute lookup returns `null` or `undefined`, the serializer finds no matching union variant and throws `invalid_union` instead of returning a null/empty string content block.

## Workaround

Use `browser_evaluate` instead:
```
browser_evaluate { expression: "document.querySelector('#el').hasAttribute('disabled').toString()" }
browser_evaluate { expression: "document.querySelector('#el').getAttribute('data-id') ?? ''" }
```

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB5
