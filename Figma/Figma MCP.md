# Figma MCP → HTML + SCSS Notes (GitHub Copilot)

## 1) Does Figma include HTML tags?

Short answer: **No (not production-ready HTML).**

Figma gives design layers (frames, text, shapes, auto-layout), not semantic web markup.  
When converting, you must create your own HTML structure and SCSS.

### Mapping guideline

- Figma frame/group → `section`, `article`, `div`, `header`, `footer` (choose semantically)
- Figma text → `h1`–`h6`, `p`, `span` (based on meaning, not appearance only)
- Figma button/input visuals → real `button`, `input`, `label`, etc.
- Figma styles (color, spacing, type, radius, shadows) → SCSS variables/tokens + component styles

---

## 2) Figma MCP in GitHub Copilot (Process, not installation)

Use MCP as **design context** inside Copilot Chat.

### Practical process

1. Open project in VS Code with Copilot Chat.
2. Ensure Figma MCP tool/context is active.
3. Prompt with:
   - Figma frame/node link
   - target stack (**plain HTML + SCSS**)
   - constraints (BEM, mobile-first, accessibility, no inline styles)
4. Ask Copilot to:
   - extract tokens first
   - then generate HTML structure
   - then SCSS
5. Review in two passes:
   - semantics/accessibility
   - visual fidelity to Figma
6. Iterate with focused prompts (small fixes only).
7. Refactor into reusable tokens/components.

---

## 3) Prompt pattern (general)

```text
Use the Figma MCP context from this frame: <FIGMA_LINK_OR_NODE_ID>.

Task:
- Build this as plain HTML + SCSS.
- First list extracted tokens (colors, type scale, spacing, radius, shadows).
- Then generate semantic, accessible markup.
- Match Auto Layout behavior with responsive CSS.

Constraints:
- BEM naming
- No inline styles
- Mobile-first
- Accessible buttons/inputs/focus states
```

---

## 4) Best workflow for plain HTML + SCSS (staged)

### Step A — HTML structure only

```text
Using Figma MCP for this frame: <FIGMA_LINK_OR_NODE_ID>,
generate only semantic HTML (no styles yet).

Requirements:
- Use landmarks: header/main/section/footer where appropriate
- Proper heading hierarchy
- Accessible form controls (label-for, button types, alt text placeholders)
- Class naming: BEM
```

### Step B — SCSS architecture scaffold

```text
Now create SCSS architecture only:
- abstracts/_variables.scss (colors, spacing, typography, radius, shadows, breakpoints)
- abstracts/_mixins.scss (media queries, focus ring)
- base/_reset.scss and base/_typography.scss
- components/<component>.scss
- layout/<section>.scss
- main.scss imports

Do not write final component styles yet; only scaffold + tokens from Figma MCP.
```

### Step C — Component/layout styling

```text
Now write SCSS for the HTML you generated.
Match Figma MCP values for spacing, type, colors, border, radius, and shadows.
Use mobile-first responsive rules.
No inline styles. No JS.
```

### Step D — Interaction states/accessibility

```text
Add :hover, :focus-visible, :active, :disabled states for interactive elements.
Ensure contrast and visible focus styles.
Keep layout unchanged.
```

### Step E — Refactor/cleanup

```text
Refactor SCSS:
- Remove duplicate values into variables
- Group repeated patterns into mixins/placeholders
- Keep specificity low (max 3 levels)
- Preserve visual output
```

---

## 5) One-shot prompt (HTML + SCSS)

```text
Use Figma MCP context from: <FIGMA_LINK_OR_NODE_ID>.

Build this in plain HTML + SCSS.

Output format:
1) HTML (semantic, accessible, BEM classes)
2) SCSS partial structure
3) SCSS code (mobile-first)
4) Notes on assumptions/assets

Rules:
- No inline styles
- No frameworks
- Use CSS variables/SCSS variables from extracted Figma tokens
- Include interaction states (:hover, :focus-visible, :disabled)
- Use responsive behavior based on Figma auto-layout, not fixed absolute positioning
```

---

## 6) What to provide Copilot for better output

- Exact frame/component link (not just file root)
- Target stack: plain HTML + SCSS
- Breakpoint behavior expectations
- Font availability
- Asset handling (SVG export vs CSS recreation)
- Accessibility requirements (focus visibility, keyboard support, contrast)

---

## 7) Common mistakes to avoid

- Prompting “convert this” without a specific frame/node
- No constraints (Copilot guesses stack/style)
- Expecting perfect production output in one shot
- Using fixed pixel positioning instead of responsive layout logic
- Skipping semantic and accessibility review

---

## 8) Quick SCSS folder structure (starter)

```text
project/
├─ index.html
└─ scss/
   ├─ abstracts/
   │  ├─ _variables.scss
   │  └─ _mixins.scss
   ├─ base/
   │  ├─ _reset.scss
   │  └─ _typography.scss
   ├─ layout/
   │  ├─ _header.scss
   │  ├─ _footer.scss
   │  └─ _sections.scss
   ├─ components/
   │  ├─ _button.scss
   │  ├─ _card.scss
   │  └─ _form.scss
   └─ main.scss
```

Example `main.scss` imports:
```scss
@use 'abstracts/variables';
@use 'abstracts/mixins';
@use 'base/reset';
@use 'base/typography';
@use 'layout/header';
@use 'layout/footer';
@use 'layout/sections';
@use 'components/button';
@use 'components/card';
@use 'components/form';
```

---

## 9) Final checklist before done

- [ ] Semantic HTML is correct
- [ ] Heading hierarchy is valid
- [ ] Form controls properly labeled
- [ ] `:focus-visible` clearly visible
- [ ] Mobile-first breakpoints match Figma behavior
- [ ] Repeated values extracted to variables/mixins
- [ ] No inline styles
- [ ] Visual match checked against frame