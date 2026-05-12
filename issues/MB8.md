# browser_screenshot — annotate: true crashes annotation script on all sites (MCP)

**vibium MCP · ChromeDriver 147.0 · macOS darwin 25.3.0**
**Filed as:** VibiumDev/vibium#156

## Summary

`browser_screenshot { annotate: true }` always fails with a script exception. The annotation script injected to label interactive elements crashes on every page tested. `annotate: false` (the default) works correctly on all sites.

## Repro

```
browser_navigate { url: "https://testtrack.org" }
browser_screenshot { annotate: true }
```

**Error:**
```
failed to annotate: script exception:
```

## Expected

Screenshot returned with numbered overlay labels on interactive elements (buttons, links, inputs).

## Actual

Script exception before screenshot is captured. No image returned.

## Scope

Confirmed on 7 sites across different frontend stacks:

| Site | Stack | annotate: true | annotate: false |
|------|-------|----------------|-----------------|
| testtrack.org | vanilla JS | script exception | ✓ works |
| coffee-cart.app | Vue SPA | script exception | ✓ works |
| saucedemo.com (post-login) | React | script exception | ✓ works |
| academybugs.com | vanilla JS | script exception | ✓ works |
| practicetestautomation.com | WordPress | script exception | ✓ works |
| the-internet.herokuapp.com | vanilla JS | script exception | ✓ works |
| demoqa.com | React/Bootstrap | script exception | ✓ works |

## Workaround

Use `annotate: false` (default). If element labeling is needed, use `browser_map` or `browser_find_all` to enumerate interactive elements separately.

## Regression skill

[lana-20/vibium-mcp-test](https://github.com/lana-20/vibium-mcp-test) — test MB8
