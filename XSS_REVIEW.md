# XSS Review: `dangerouslySetInnerHTML` Audit

This document catalogs every use of `dangerouslySetInnerHTML` in the codebase, along with a risk assessment and remediation recommendation for each.

## Active (`apps/v4`) Usages

### Code Highlighting

These usages render HTML produced by the **Shiki** syntax highlighter on the server side (`apps/v4/lib/highlight-code.ts`). The input is source code read from the filesystem or registry, and the output is Shiki-generated `<pre>/<code>/<span>` markup.

| Code Pointer | Usage | Risk | shouldPR |
|---|---|---|---|
| `apps/v4/components/chart-code-viewer.tsx:72` | Renders `chart.highlightedCode` — Shiki-highlighted source of a chart component, displayed in a code viewer sheet/drawer. | Low. The input is server-side source code from the registry, processed by Shiki. If the registry data source were compromised or a malicious registry item were added, the Shiki output could contain injected HTML. However, Shiki escapes code content by default. | No |
| `apps/v4/components/block-viewer.tsx:363` | Renders `file?.highlightedContent` — Shiki-highlighted source for the active file in the block code viewer. | Low. Same data flow as above: registry file content → Shiki → HTML. Protected by Shiki's escaping. | No |
| `apps/v4/components/component-source.tsx:113` | Renders `highlightedCode` — Shiki-highlighted source for component documentation pages. Source is read from filesystem (`fs.readFile`) or fetched from the registry. | Low. Input is controlled server-side source code. Shiki escapes content. Risk only if a build-time supply-chain attack injected malicious content into source files. | No |

### Inline Scripts (Hardcoded)

These usages inject **hardcoded JavaScript strings** into `<script>` tags. No dynamic or user-supplied data flows into the HTML string.

| Code Pointer | Usage | Risk | shouldPR |
|---|---|---|---|
| `apps/v4/app/layout.tsx:78` | Injects a `<script>` in `<head>` that reads `localStorage.theme` and sets the `meta[name="theme-color"]` attribute and a layout class to prevent flash of unstyled content (FOUC). | None. The JS string is a hardcoded template literal with only build-time constants (`META_THEME_COLORS.dark`). No user input is interpolated. | No |
| `apps/v4/components/mode-switcher.tsx:94` | Injects a `<Script>` (`next/script`, `beforeInteractive`) that listens for the `D` keypress and forwards a `postMessage` to the parent frame for dark mode toggling. | None. Entirely hardcoded JS. The `postMessage` uses a constant type string. No user input. | No |
| `apps/v4/app/(create)/components/random-button.tsx:152` | Injects a `<Script>` that listens for the `R` keypress and forwards a `postMessage` to the parent frame for randomization. | None. Entirely hardcoded JS with a constant type. No user input. | No |
| `apps/v4/app/(create)/components/item-picker.tsx:172` | Injects a `<Script>` that listens for `Cmd/Ctrl+K` and `Cmd/Ctrl+P` keypresses and forwards them via `postMessage` to the parent frame. | None. Entirely hardcoded JS with a constant type. No user input. | No |

### SVG Logo Rendering

These usages render SVG markup for logos. The SVG strings come from either a static JSON file (`registry/directory.json`) or from hardcoded configuration objects (`BASES`, `TEMPLATES`).

| Code Pointer | Usage | Risk | shouldPR |
|---|---|---|---|
| `apps/v4/components/directory-list.tsx:44` | Renders `registry.logo` — an SVG string from `registry/directory.json`, displayed as a logo in the component directory listing. | Medium. The `directory.json` file is committed to the repo and contains SVG strings for ~100+ third-party registries. A malicious SVG submitted via PR (e.g., containing `<script>`, `onload` handlers, or `<foreignObject>` with embedded HTML) could execute arbitrary JS. SVG content is not sanitized before rendering. | Yes |
| `apps/v4/components/docs-base-switcher.tsx:32` | Renders `activeBase.meta.logo` — an SVG string from the base configuration, shown in the docs base switcher tab bar. | Low. The base configs are defined in code within the repo. An attacker would need to compromise the repo itself. However, the SVG is still unsanitized. | No |
| `apps/v4/app/(create)/components/template-picker.tsx:56` | Renders `currentTemplate.logo` — SVG for the currently selected template, shown in the picker trigger button. | Low. Template configs are hardcoded in source. SVG is not sanitized but the source is trusted. | No |
| `apps/v4/app/(create)/components/template-picker.tsx:81` | Renders `template.logo` — SVG for each template option in the picker dropdown. | Low. Same static config source as above. | No |
| `apps/v4/app/(create)/components/base-picker.tsx:54` | Renders `currentBase.meta.logo` — SVG for the currently selected base library, shown in the picker trigger. | Low. Base configs are hardcoded in source. | No |
| `apps/v4/app/(create)/components/base-picker.tsx:75` | Renders `base.meta.logo` — SVG for each base option in the picker dropdown. | Low. Same static config source as above. | No |
| `apps/v4/app/(create)/components/toolbar-controls.tsx:203` | Renders `template.logo` — SVG for each template option in the toolbar radio group. | Low. Same static template config source. | No |

### Chart CSS Style Injection

These usages inject a `<style>` tag with dynamically constructed CSS custom properties for chart theming. The `id` and color values from `ChartConfig` are interpolated directly into the CSS string without sanitization.

| Code Pointer | Usage | Risk | shouldPR |
|---|---|---|---|
| `apps/v4/registry/new-york-v4/ui/chart.tsx:83` | `ChartStyle` component: builds a `<style>` tag with CSS custom properties (`--color-{key}: {color}`) scoped via `[data-chart={id}]`. The `id` is derived from either a user-provided `id` prop or `React.useId()`, and `color` values come from `ChartConfig`. | Medium. The `id` prop on `ChartContainer` is a public API — any consumer can pass an arbitrary string. If user-controlled input reaches `id` or `color`/`theme` values, an attacker could break out of the CSS selector or property value to inject arbitrary CSS (e.g., exfiltrating data via `background: url(...)` or defacing the page). In practice, all current usages use hardcoded configs, but since this is a **library component distributed to end users**, consumers may unknowingly pass unsanitized input. The `id` is not quoted in the CSS selector (`[data-chart=${id}]` instead of `[data-chart="${id}"]`), making breakout trivial. | Yes |
| `apps/v4/registry/bases/base/ui/chart.tsx:83` | Identical `ChartStyle` pattern — base variant. | Medium. Same issue as above. | Yes |
| `apps/v4/registry/bases/radix/ui/chart.tsx:83` | Identical `ChartStyle` pattern — radix variant. | Medium. Same issue as above. | Yes |
| `apps/v4/examples/base/ui/chart.tsx:82` | Identical `ChartStyle` pattern — base example. | Medium. Same issue as above. | Yes |
| `apps/v4/examples/base/ui-rtl/chart.tsx:82` | Identical `ChartStyle` pattern — base RTL example. | Medium. Same issue as above. | Yes |
| `apps/v4/examples/radix/ui/chart.tsx:82` | Identical `ChartStyle` pattern — radix example. | Medium. Same issue as above. | Yes |
| `apps/v4/examples/radix/ui-rtl/chart.tsx:82` | Identical `ChartStyle` pattern — radix RTL example. | Medium. Same issue as above. | Yes |

## Deprecated (`deprecated/www`) Usages

These mirror the active usages above but live in the deprecated `www` app. They carry the same risks.

| Code Pointer | Usage | Risk | shouldPR |
|---|---|---|---|
| `deprecated/www/components/chart-code-viewer.tsx:118` | Renders `chart.highlightedCode` — Shiki output for chart source in a code viewer. | Low. Same pattern as the active equivalent. Server-side Shiki output. | No |
| `deprecated/www/components/block-viewer.tsx:305` | Renders `file?.highlightedContent` — Shiki output for block file viewer. | Low. Same pattern as the active equivalent. | No |
| `deprecated/www/app/layout.tsx:81` | Hardcoded `<script>` for theme initialization (FOUC prevention). | None. Hardcoded JS with build-time constants only. | No |
| `deprecated/www/registry/new-york/ui/chart.tsx:81` | `ChartStyle` — same CSS injection pattern as the active chart components. | Medium. Same unquoted `id` and unsanitized color interpolation issue. However, this is deprecated code. | No |
| `deprecated/www/registry/default/ui/chart.tsx:81` | `ChartStyle` — same CSS injection pattern. | Medium. Same issue. Deprecated code. | No |

## Summary

| Risk Level | Count | shouldPR |
|---|---|---|
| None (hardcoded inline scripts) | 4 | 0 |
| Low (Shiki output / trusted static config) | 11 | 0 |
| Medium (unsanitized SVG from directory.json) | 1 | 1 |
| Medium (CSS injection via ChartStyle — active) | 7 | 7 |
| Medium (CSS injection via ChartStyle — deprecated) | 2 | 0 |
| **Total** | **25** | **8** |

## Recommended Remediations

### 1. ChartStyle CSS Injection (7 active files)

**Problem:** The `id` value is interpolated unquoted into a CSS attribute selector, and `color` values are interpolated directly into CSS property values. Neither is sanitized.

**Fix:**
- Quote the `id` in the CSS selector: `[data-chart="${id}"]`
- Sanitize `id` to only allow `[a-zA-Z0-9_-]` characters
- Sanitize `color` values to only allow valid CSS color formats (hex, rgb, hsl, CSS variables)

### 2. Directory SVG Logos (1 file)

**Problem:** SVG strings from `registry/directory.json` are rendered without sanitization. This file grows via community PRs and could contain malicious SVG.

**Fix:**
- Sanitize SVG strings using a library like [DOMPurify](https://github.com/cure53/DOMPurify) before rendering, or
- Validate SVGs at PR review time with an automated check that rejects `<script>`, event handler attributes (`on*`), and `<foreignObject>` elements, or
- Convert logos to static image files or use an allowlist of safe SVG elements/attributes
