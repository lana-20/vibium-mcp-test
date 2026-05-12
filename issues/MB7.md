# browser_fill — fails on <textarea> elements (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#155

## Summary

`browser_fill` always fails with `failed to fill:` when the target element is a `<textarea>`. It works correctly on `<input>` elements. The BiDi fill implementation does not handle the textarea element type.

## Repro

```
browser_set_content { html: "<html><body><textarea id='ta'></textarea></body></html>" }
browser_fill { selector: "#ta", text: "hello textarea" }
```

**Error:**
```
failed to fill: fill:
```

## Expected

Clears the textarea and types the specified text, consistent with behavior on `<input>` elements.

## Actual

Fails immediately with `failed to fill:` for any textarea selector, on any page.

## Scope

Confirmed on 6 sites across injected and real-world textareas:

| Site | Selector | Result |
|------|----------|--------|
| injected HTML | `#ta` | `failed to fill:` |
| testpages.eviltester.com (basic HTML form) | `textarea` | `failed to fill:` |
| automationintesting.online | `textarea#description` | `failed to fill:` |
| testtrack.org/text-input-demo | `textarea` | `failed to fill:` |
| practice-automation.com/form-fields | `textarea` | `failed to fill:` |
| demoqa.com/text-box | `#currentAddress` | `failed to fill:` |
| testautomationpractice.blogspot.com | `textarea` | `failed to fill:` |

## Workaround

Use `browser_click` to focus the element, then `browser_type` to enter text:
```
browser_click { selector: "#ta" }
browser_type { selector: "#ta", text: "hello textarea" }
```

Alternatively, use `browser_evaluate` to set the value and dispatch an input event:
```
browser_evaluate { expression: "const el = document.querySelector('#ta'); el.value = 'hello textarea'; el.dispatchEvent(new Event('input', {bubbles: true}))" }
```

## See also

- #117 — same bug in the CLI (`vibium fill`); includes Go-level root cause (`buildSetValueScript` calls `HTMLInputElement.prototype.set` on a textarea → `Illegal invocation`)

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB7
