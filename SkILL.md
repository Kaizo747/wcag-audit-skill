---
name: accessibility-audit
description: Run accessibility audits on web projects against WCAG 2.2 (Level AA or AAA) using axe-core for runtime browser scanning and eslint-plugin-jsx-a11y for static JSX/TSX analysis. Use this whenever the user asks to run an accessibility scan, a11y audit, WCAG compliance check, or invokes /accessibility or /a11y — including requests about specific concerns like color contrast, alt text, keyboard focus, target/touch size, ARIA, screen reader support, or Section 508/ADA/EAA compliance — even if they don't name a scan mode or conformance level.
---

# Accessibility Audit Skill

Run comprehensive accessibility audits on web projects using axe-core (runtime browser scanning) and eslint-plugin-jsx-a11y (static analysis), evaluated against **WCAG 2.2** — the current W3C Recommendation since October 2023, and the version now referenced by the EU Accessibility Act, evolving Section 508 guidance, and most WCAG-based policy. Configurable scan modes and conformance levels let you match the depth of the audit to what's actually needed.

This isn't a legal compliance certification. Automated tools plus the manual checklist below catch a large share of real issues, but a genuine conformance claim also needs testing with real assistive technology (screen readers, switch access, voice control) — ideally by people who use it daily.

## Configuration

Two independent choices shape the audit. Ask the user for both if they aren't specified, but don't block on it — `runtime` scan mode at Level `AA` is a sensible default if they just want to get moving.

### Scan mode

1. **`runtime`** (default) — Browser-based axe-core scan
   - Injects axe-core into running pages via browser automation
   - Scans multiple pages if a sitemap or route list is available
   - Reports violations with impact level, affected elements, and fix recommendations

2. **`static`** — ESLint jsx-a11y static analysis
   - Runs eslint-plugin-jsx-a11y rules against source code
   - Catches issues at build time without needing a running server
   - Covers React/Next.js/Preact/Solid-style JSX and TSX — **not** Vue single-file components (see the Static Analysis section)

3. **`full`** — Both runtime + static combined
   - Runs static analysis first for fast feedback
   - Then starts dev server and runs axe-core on all discoverable pages
   - Provides the most comprehensive automated coverage; pair with the manual checklist below for genuine WCAG 2.2 coverage

### Conformance level

1. **`AA`** (default) — matches nearly every real-world mandate that cites WCAG (EAA, ADA Title II, Section 508, most public-sector policy)
2. **`AAA`** — adds axe-core's automatable WCAG AAA rules (e.g. enhanced 7:1 contrast) plus the AAA-only rows in the manual checklist below. Mention to the user that the W3C itself doesn't recommend requiring full-site AAA conformance as blanket policy — some content genuinely can't satisfy every AAA criterion regardless of how it's built, so AAA is normally scoped to specific pages, not declared site-wide.

## WCAG 2.2 Quick Reference

WCAG 2.2 became a W3C Recommendation on 5 October 2023. It's backward-compatible with 2.0 and 2.1 except for one removed criterion — **4.1.1 Parsing** is obsolete in 2.2, since modern browsers and assistive tech now handle malformed HTML robustly enough that it stopped being a meaningful conformance concern. Don't flag a missing 4.1.1 check as a gap; that's expected.

It adds nine new success criteria:

| SC | Name | Level | Automatable? |
|----|------|-------|---------------|
| 2.4.11 | Focus Not Obscured (Minimum) | AA | Manual |
| 2.4.12 | Focus Not Obscured (Enhanced) | AAA | Manual |
| 2.4.13 | Focus Appearance | AAA | Manual |
| 2.5.7 | Dragging Movements | AA | Manual |
| 2.5.8 | Target Size (Minimum) | AA | Yes — axe-core's `target-size` rule |
| 3.2.6 | Consistent Help | A | Manual |
| 3.3.7 | Redundant Entry | A | Manual |
| 3.3.8 | Accessible Authentication (Minimum) | AA | Manual |
| 3.3.9 | Accessible Authentication (Enhanced) | AAA | Manual |

Target Size is the only one of the nine that's reliably automatable — Deque (axe-core's maintainer) has said this is likely the only WCAG 2.2 criterion they'll ever add a rule for, since the rest need human judgment to avoid false positives. That's the reason the Manual Verification Checklist section exists: without it, a "WCAG 2.2 audit" is really just a WCAG 2.1 audit with one extra rule.

## Runtime Scan Procedure

### Step 1: Discover pages to scan
Look for routes in the project:
- Next.js: Check `src/app/**/page.tsx` or `pages/**/*.tsx` for routes
- React Router: Check route definitions
- Static sites: Check HTML files or sitemap.xml
- Ask user if unclear

### Step 2: Start dev server
- Detect the project's dev command from `package.json` scripts (usually `dev`, `start`, or `serve`)
- Start it in the background
- Wait for HTTP 200 on localhost

### Step 3: Inject axe-core and scan each page

Don't reuse an old pinned version number across audits. Axe-core ships a new minor release every 3-5 months, usually with new rules — the `target-size` WCAG 2.2 rule itself only exists from v4.5.0 onward, so a stale pin can silently under-report. Check the current release first (`npm view axe-core version`, or https://github.com/dequelabs/axe-core/releases) and inject that version. If the project already depends on `axe-core` (check `package.json` — common alongside `@axe-core/playwright` or `@axe-core/react`), prefer reading and injecting that local copy instead of the CDN, so results match whatever version the team's own tests already use.

For each page, use browser automation to:
1. Navigate to the page
2. Wait for content to load (2-3 seconds)
3. Inject axe-core from CDN (substitute the current version):
```javascript
https://cdnjs.cloudflare.com/ajax/libs/axe-core/<CURRENT_VERSION>/axe.min.js
```
4. Run the audit, choosing tags for the conformance level selected in Configuration:
```javascript
(async () => {
  if (!window.axe) {
    const script = document.createElement('script');
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/<CURRENT_VERSION>/axe.min.js';
    document.head.appendChild(script);
    await new Promise(resolve => { script.onload = resolve; });
  }
  const results = await axe.run(document, {
    // AA (default): wcag2a, wcag2aa, wcag21a, wcag21aa, wcag22aa, best-practice
    // AAA: also add 'wcag2aaa' — axe-core has no wcag21aaa/wcag22aaa tag yet,
    //      so AAA-only 2.2 criteria are covered by the manual checklist instead
    runOnly: ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa', 'best-practice']
  });
  window.__axeResults = {
    page: window.location.pathname,
    violations: results.violations.length,
    passes: results.passes.length,
    incomplete: results.incomplete.length,
    details: results.violations.map(v => ({
      id: v.id,
      impact: v.impact,
      description: v.description,
      helpUrl: v.helpUrl,
      nodes: v.nodes.length,
      elements: v.nodes.slice(0, 5).map(n => ({
        html: n.html.substring(0, 200),
        target: n.target,
        failureSummary: n.failureSummary
      }))
    }))
  };
})().then(() => console.log('AXE_SCAN_DONE'));
```
5. Wait 3 seconds, then retrieve `window.__axeResults`

Check the `incomplete` array as well as `violations` — axe-core's `target-size` rule frequently returns a "needs review" result rather than a clean pass/fail (e.g. when it can't confirm the spacing exception applies), and those items still belong in the report.

### Step 4: Kill dev server
After all pages are scanned, kill the background dev server process.

### Step 5: Generate report
Format results as a markdown table with:
- Page path
- Number of violations, needs-review items, and passes
- Impact levels

Then list each violation with:
- Rule ID and description
- Impact (critical/serious/moderate/minor)
- Affected elements (HTML snippet + CSS selector)
- Fix recommendation
- WCAG reference link

Append the Manual Verification Checklist results (see below) — filled in if the checks were actually performed, or left as an action-item list if not.

## Static Analysis Procedure

### Step 1: Check if eslint-plugin-jsx-a11y is installed
Look in `package.json` devDependencies. This plugin does a static evaluation of JSX — it covers React, Next.js, Preact, Solid, and similar JSX/TSX ecosystems, but it does **not** apply to Vue single-file components: `<template>` blocks aren't JSX, so jsx-a11y rules simply won't fire on `.vue` files. For a Vue project, use `eslint-plugin-vuejs-accessibility` instead (it runs on `vue-eslint-parser`'s template AST) — don't run jsx-a11y against Vue templates and report a clean pass, since that would mean nothing actually got checked.

If not installed:
```bash
pnpm add -D eslint-plugin-jsx-a11y  # or npm/yarn based on lockfile
```

### Step 2: Create a temporary standalone config
To avoid disturbing the project's own ESLint setup — which may pin a different parser or rule set this audit shouldn't touch — scope the a11y rules to an isolated config file used only for this run:
```javascript
// eslint.a11y.mjs
import jsxA11y from "eslint-plugin-jsx-a11y";

export default [
  {
    files: ["src/**/*.jsx", "src/**/*.tsx"],
    plugins: { "jsx-a11y": jsxA11y },
    languageOptions: {
      parserOptions: { ecmaFeatures: { jsx: true } },
    },
    settings: {
      "jsx-a11y": {
        // Map custom/design-system components to the HTML element they
        // render, e.g. { Button: "button", Link: "a" } — otherwise
        // jsx-a11y can't see through them and will under- or over-report.
        components: {},
      },
    },
    rules: { ...jsxA11y.flatConfigs.recommended.rules },
  },
];
```
If the project is TypeScript, swap in `typescript-eslint`'s parser so TSX parses correctly:
```javascript
import tseslint from "typescript-eslint";
// use in place of the languageOptions above:
languageOptions: {
  parser: tseslint.parser,
  parserOptions: { ecmaFeatures: { jsx: true } },
},
```
Plain JS/JSX projects can skip this — ESLint's default parser already handles JSX once `ecmaFeatures.jsx` is set.

### Step 3: Run ESLint with the a11y config
```bash
npx eslint --config eslint.a11y.mjs src/
```

### Step 4: Report findings
List each violation with file, line number, rule, and description. Note in the report that this pass only reflects WCAG 2.0/2.1-era concerns (semantic HTML, ARIA validity, labels, alt text) — jsx-a11y has no rules targeting the WCAG 2.2 criteria specifically, since almost all of them are rendered/behavioral (focus, dragging, timing) rather than visible in a JSX AST. The runtime scan and manual checklist are what actually cover 2.2.

### Step 5: Clean up
Remove the temporary config file. Keep `eslint-plugin-jsx-a11y` installed for future use.

### Step 6: Handle false positives
Common false positives to note:
- Custom component props named `role` (not HTML role attribute)
- Components that pass through ARIA attributes to child elements
- Dynamic content loaded after initial render
- Custom design-system components (e.g. `<Button>`, `<Link>`) that jsx-a11y can't see through — map them in `settings["jsx-a11y"].components` (shown in Step 2) rather than suppressing the rule outright

## Manual Verification Checklist (WCAG 2.2)

Walk through each row that applies to the chosen conformance level. These can't be reliably automated, so they need an actual person (or a guided pass through the running app) rather than a script.

| SC | Name | Level | Quick check |
|----|------|-------|--------------|
| 2.4.11 | Focus Not Obscured (Minimum) | AA | Tab through every interactive element; confirm no sticky header, footer, or cookie banner ever fully hides the focused item |
| 2.4.12 | Focus Not Obscured (Enhanced) | AAA | Same pass, but the focused item must be *completely* clear, not just partially visible |
| 2.4.13 | Focus Appearance | AAA | Tab to a control and confirm the focus indicator is at least as thick as a 2px CSS-pixel perimeter around it, with roughly 3:1 contrast against its surroundings |
| 2.5.7 | Dragging Movements | AA | Find every drag interaction (reorderable lists, sliders, map pins); confirm each has a single-click/tap alternative (e.g. up/down or "move to top" buttons) |
| 2.5.8 | Target Size (Minimum) | AA | Already covered by the automated `target-size` rule above — spot-check icon-only buttons and tightly packed links if it's out of scope for this run |
| 3.2.6 | Consistent Help | A | If a help/contact/chat mechanism appears on multiple pages, confirm it's always in the same relative position and order |
| 3.3.7 | Redundant Entry | A | Walk a multi-step form or checkout; confirm information entered earlier isn't asked for again without being pre-filled or selectable |
| 3.3.8 | Accessible Authentication (Minimum) | AA | Confirm login never requires solving a puzzle or re-typing a code from memory without an alternative — pasting and password managers must work |
| 3.3.9 | Accessible Authentication (Enhanced) | AAA | Confirm login never requires object recognition (e.g. "select all traffic lights") or recalling personal content, with no exceptions |

## Auto-Fix Suggestions

For each violation found, provide specific fix recommendations:

| Violation | Fix |
|-----------|-----|
| `region` (content not in landmarks) | Wrap page content in `<main>`, ensure nav in `<nav>`, footer in `<footer>` |
| `heading-order` (skipped heading levels) | Change heading level or add intermediate headings; use `<p>` with styling for decorative text |
| `color-contrast` | Adjust foreground/background colors to meet 4.5:1 ratio (3:1 for large text; 7:1 for AAA) |
| `image-alt` | Add descriptive `alt` text; use `alt=""` for decorative images |
| `button-name` | Add text content, `aria-label`, or `aria-labelledby` to buttons |
| `link-name` | Add text content or `aria-label` to links |
| `label` | Associate `<label>` with form controls via `htmlFor`/`id` or wrapping |
| `aria-*` | Fix invalid ARIA attributes, roles, or values |
| `target-size` | Grow the interactive element to at least 24x24 CSS px, or add enough spacing that a 24px circle centered on it won't overlap a neighboring target |
| Focus Not Obscured *(manual)* | Add `scroll-margin-top`/`scroll-padding-top` matching sticky header/footer height so focus scrolls clear of overlays |
| Focus Appearance *(manual)* | Strengthen `:focus-visible` to a >=2px outline with >=3:1 contrast; never ship `outline: none` without a replacement |
| Dragging Movements *(manual)* | Add a non-drag alternative next to any drag interaction |
| Consistent Help *(manual)* | Keep help/contact links in the same DOM order and position across pages |
| Redundant Entry *(manual)* | Auto-populate or offer a "same as above" shortcut instead of re-asking |
| Accessible Authentication *(manual)* | Allow paste and password managers; avoid CAPTCHA-only flows; support magic links or paste-friendly OTPs |

## Output Format

Always present the final report in this structure — include one row in the Manual Verification Checklist table per criterion that applies to the chosen conformance level:

````
## Accessibility Audit Results

**Scan mode:** [runtime/static/full]
**Standards:** WCAG 2.2 Level [AA/AAA] + Best Practices
**Pages scanned:** [count]

| Page | Violations | Needs Review | Passes | Worst Impact |
|------|-----------|--------------|--------|--------------|
| /    | 2         | 1            | 38     | moderate     |

### Automated Violations

#### 1. [rule-id] — [impact]
**Description:** [what the rule checks]
**Pages affected:** [list]
**Elements:** [count] instances
**Example:**
```html
<element that failed>
```
**Fix:** [specific recommendation]
**WCAG:** [reference]

---

### Manual Verification Checklist
| SC | Name | Result | Notes |
|----|------|--------|-------|
| 2.4.11 | Focus Not Obscured (Minimum) | [Pass/Fail/Not checked] | [notes] |
| ...    | ...                          | ...                     | ...    |

---

### Summary
- Total automated violations: [n]
- Critical: [n] | Serious: [n] | Moderate: [n] | Minor: [n]
- Manual items needing follow-up: [n]
- Recommendation: [prioritized next steps]
````

## Resources

- WCAG 2.2 Quick Reference (official, filterable): https://www.w3.org/WAI/WCAG22/quickref/
- What's New in WCAG 2.2 (W3C): https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/
- axe-core rule descriptions: https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md
- axe-core releases (check the current version before injecting): https://github.com/dequelabs/axe-core/releases
- eslint-plugin-jsx-a11y: https://github.com/jsx-eslint/eslint-plugin-jsx-a11y
- eslint-plugin-vuejs-accessibility (for Vue SFCs — not covered by jsx-a11y): https://vue-a11y.github.io/eslint-plugin-vuejs-accessibility/
