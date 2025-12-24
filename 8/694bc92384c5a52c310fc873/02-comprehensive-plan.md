### 1. CODEBASE ANALYSIS

The repo’s theming mechanism is unknown (reference files couldn’t be read in the planning context), so the implementation must start by **identifying the authoritative “core colors/tokens” source** and then ensuring **all other theming layers (Tailwind, component library theme objects, global CSS)** consume those tokens without divergence.

**What to determine up front (decisive branching):**
1. **Is the source of truth CSS variables?**  
   Likely in `src/styles/theme.css` (or `globals.css`) with tokens like `--color-primary`, `--color-link`, `--ring`, etc.
2. **Is the source of truth a TS theme object?**  
   Likely in `src/theme.ts` or `src/theme/index.ts` (MUI/Chakra/styled-components/etc.) with `palette.primary`, `components` overrides, etc.
3. **Do both exist? If yes, which is authoritative?**  
   - If Tailwind + CSS variables exist and components use utility classes → **CSS variables are likely authoritative**.  
   - If a ThemeProvider is used broadly and components read from it → **TS theme object is likely authoritative**.  
   - If both are used, pick **one** as the source of truth and make the other **derive from it** (e.g., TS theme reads CSS vars, or CSS vars are generated from TS tokens) to avoid mismatched “primary” values.

**Dark mode detection to identify:**
- `.dark` class, `[data-theme="dark"]`, `@media (prefers-color-scheme: dark)`, or separate theme objects (e.g., `lightTheme`/`darkTheme`).

**Token coverage to explicitly confirm exists (or add if missing):**
- `primary/brand/accent` (base)
- `primary-hover`
- `primary-active`
- `primary-disabled` (or disabled treatment rules)
- `on-primary` (text/icon on primary surfaces)
- `link` and `link-hover` (often tied to primary)
- `focus-ring` / `ring` / `outline` token(s) used by `:focus-visible`
- Any “primary button background” token if separate from `primary`

This analysis step ensures we update **exactly the centralized tokens** required by the task: **primary/accent, link color, primary button background, focus/active states**, and do so for **both light/dark** if supported.

---

### 2. IMPLEMENTATION STRATEGY

1. **Find and declare the single source of truth** for core colors (CSS variables file or TS theme tokens).
2. **Update primary/brand tokens to a red palette** (any reasonable red shade) including **hover/active/disabled/focus** tokens, and ensure **link** and **primary button background** are tied to these tokens (not hardcoded).
3. **Update both light and dark palettes** using the repo’s actual dark-mode mechanism (class/data attribute/media query or separate theme objects).
4. **Ensure Tailwind mappings (if used) and component library theme mappings (if used) point to the same tokens**, including any **scale-based utilities** (e.g., `primary-50..900`) if the codebase relies on them.
5. **Repo-wide search and cleanup**: remove/replace any lingering hardcoded old brand color values in components with token references.
6. **Manual verification sweep** focused on the required surfaces: **buttons, links, nav active states, headers, form focus rings, alerts**, in both light/dark modes.
7. **Accessibility/contrast checks**: explicitly verify contrast for (a) primary button text on red background and (b) link text on page background.

(Per instruction: **do not add unit tests here**; tests are handled by a separate node.)

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Identify the authoritative core token source and dark-mode mechanism**
- **File(s)**:  
  `src/styles/theme.css`, `src/styles/globals.css`, `src/theme.ts`, `src/theme/index.ts`, `src/App.tsx`, `src/main.tsx`, `tailwind.config.js|ts`
- **Action**: Determine where primary/link/focus tokens are defined and how dark mode is applied.
- **Details**:
  - Search for: `--color-primary`, `--brand`, `--accent`, `--link`, `--ring`, `focus`, `outline`, `palette.primary`, `primary.main`, `ThemeProvider`, `.dark`, `data-theme`.
  - Decide **one** source of truth:
    - If CSS variables exist and are referenced widely → treat `theme.css` as authoritative.
    - If TS theme object is used by a provider and components rely on it → treat `src/theme.ts` as authoritative.
    - If both exist → choose one and make the other **derive** from it (avoid two independent “primary” definitions).
- **Rationale**: Prevents partial updates and mismatched primary colors across systems.
- **Addresses**: Global propagation requirement; validation rule #1; dark mode coverage.

---

**Step 2: Update light-mode primary/brand/accent + link + primary-button-bg tokens to red (core token source)**
- **File**: the authoritative core token file identified in Step 1 (likely `src/styles/theme.css` or `src/theme.ts`)
- **Action**: Set primary/brand/accent to a red hue and ensure all required related tokens are updated.
- **Details** (adapt names to repo conventions; ensure each exists and is used):
  - `primary` (base): choose a red with good contrast for white text (e.g., a “600–700” style red).
  - `primary-hover`: darker than base.
  - `primary-active`: darker than hover.
  - `primary-disabled`: clearly distinct (often reduced saturation + higher lightness, or opacity rules).
  - `on-primary`: typically `#fff` (or near-white) for readable text/icons.
  - `link`: red aligned with primary (may be slightly darker for readability on white).
  - `link-hover`: darker than link.
  - `primary-button-bg`: if separate token exists, point it to `primary` (or set it to the same red).
  - `focus-ring`: define a ring color that remains visible on white backgrounds (often a lighter red/pink with alpha).
- **Rationale**: Explicitly covers all required tokens: primary/accent, link color, primary button background, focus/active states.
- **Addresses**: Acceptance criteria 1–3; validation rules #1–4.

---

**Step 3: Update dark-mode palette tokens to red using the repo’s actual dark-mode selector/object**
- **File**: same authoritative token source as Step 2
- **Action**: Update dark-mode primary/link/focus tokens with red shades appropriate for dark surfaces.
- **Details**:
  - If CSS variables:
    - Update within `.dark { ... }`, `[data-theme="dark"] { ... }`, or `@media (prefers-color-scheme: dark)` block—whichever the repo uses.
  - If TS theme objects:
    - Update `darkTheme.palette.primary` (and link/focus tokens if present).
  - Dark-mode red often needs to be **slightly brighter** than light-mode base to read well on dark UI.
  - Ensure `on-primary` remains readable (near-white) and focus ring is visible on dark backgrounds.
- **Rationale**: Ensures branding consistency and readability in dark mode without missing the correct selector/theme object.
- **Addresses**: Acceptance criterion 4; validation rules #3–5.

---

**Step 4: Ensure focus visibility on red surfaces via a concrete ring strategy (avoid blending)**
- **File(s)**: authoritative token source + `src/styles/globals.css` (or equivalent global focus styles)
- **Action**: Implement/confirm a focus ring that is visible both on neutral backgrounds and on red primary buttons.
- **Details**:
  - Prefer a **dual-ring/offset** approach if the system supports it:
    - Inner ring: light/white-ish outline (or background-colored offset)
    - Outer ring: red/pink ring token
  - Ensure `:focus-visible` styles reference tokens (e.g., `--focus-ring`, `--focus-ring-offset`, `--focus-ring-width`).
  - Confirm buttons/links/inputs do not remove outline without replacement.
- **Rationale**: Makes acceptance criterion #3 robust, especially when the element itself is red.
- **Addresses**: Acceptance criterion 3; validation rule #4.

---

**Step 5: Tailwind mapping update (if Tailwind is used), including scale-based utilities**
- **File**: `tailwind.config.js` or `tailwind.config.ts`
- **Action**: Ensure Tailwind `primary/brand/accent/link/ring` colors map to the centralized tokens and that any required color scale utilities still work.
- **Details**:
  - If the codebase uses `text-primary`, `bg-primary`, `ring-primary`, map those to CSS variables or theme tokens.
  - If the codebase uses a scale like `primary-50 ... primary-900`:
    - Either map each scale step to corresponding tokens/variables (preferred if already used), or
    - Provide a minimal scale that preserves expected steps used in the app (audit usage via search for `primary-` classes).
  - Ensure hover/active utilities reference distinct tokens (e.g., `bg-primary-hover`, `bg-primary-active`) or scale steps that differ.
- **Rationale**: Prevents Tailwind utilities from retaining old colors or breaking if the app expects a scale.
- **Addresses**: Acceptance criteria 1–2; validation rules #1–2.

---

**Step 6: Component library theme mapping (if present) to ensure semantic + primary separation**
- **File(s)**: `src/theme.ts`, `src/theme/index.ts` (or equivalent)
- **Action**: Ensure the component library’s “primary” uses the updated red tokens and semantic colors remain unchanged.
- **Details**:
  - Update `palette.primary` (or equivalent) to red.
  - Ensure `components` overrides for Button/Link/Focus states reference `primary` tokens.
  - Explicitly verify `success/warning/error` palettes are not derived from `primary`.
  - Check for mappings like `Alert` variants or “status” components that might incorrectly use `primary`—correct them to semantic tokens if they are wrongly tied.
- **Rationale**: Avoids accidental recoloring of semantic UI and ensures library components follow the new brand color.
- **Addresses**: Acceptance criteria 1–2, 5; validation rule #6.

---

**Step 7: Repo-wide search for old brand color hardcoding and replace with tokens**
- **File(s)**: across `src/**` (components, pages, styles)
- **Action**: Remove lingering hardcoded old brand colors and ensure components reference tokens/utilities.
- **Details**:
  - Search for:
    - Previous brand hex values (once identified from current tokens)
    - Common blues/purples used as old primary (e.g., `#2563EB`, `#3B82F6`, etc.)—but only replace when clearly “brand primary”.
    - `rgb(...)` equivalents and named colors if used.
  - Replace with:
    - CSS variables (`var(--color-primary)`, etc.), or
    - Theme references (`theme.palette.primary.main`), or
    - Tailwind utilities that map to tokens (`bg-primary`, `text-link`, `ring-primary`).
- **Rationale**: Ensures global propagation and prevents “one-off” components from staying on the old color.
- **Addresses**: Acceptance criteria 1–2; validation rule #1.

---

**Step 8: Verify key UI surfaces explicitly (nav, headers, buttons, links, forms, alerts) are token-driven**
- **File(s)**: likely nav/header components, button/link components, form components, alert components
- **Action**: Confirm these surfaces consume the updated tokens and don’t override them locally.
- **Details**:
  - **Nav/headers**: active item indicator, selected tab underline, header action buttons.
  - **Buttons**: primary background, hover/active/disabled, text color.
  - **Links**: default + hover + focus-visible.
  - **Forms**: input focus ring/border, checkbox/radio checked state if tied to primary.
  - **Alerts**: success/warning/error remain semantic; only “info/primary” (if exists) should be red.
- **Rationale**: Matches the task’s explicitly called-out surfaces and ensures no regressions.
- **Addresses**: Acceptance criteria 1–5; validation rules #1–6.

---

**Step 9: Ensure interaction states are distinct via explicit token differences**
- **File**: authoritative token source
- **Action**: Confirm hover/active/disabled tokens are not identical and are used by components.
- **Details**:
  - Ensure `primary-hover` ≠ `primary` and `primary-active` ≠ `primary-hover`.
  - Ensure disabled state is visually distinct (either via separate token or consistent opacity + cursor + text color rules).
  - Ensure link hover differs from link default.
- **Rationale**: Makes acceptance criterion #2 verifiable and repeatable.
- **Addresses**: Acceptance criterion 2; validation rule #2.

---

### 4. VALIDATION STRATEGY

**A. Token-level validation (source of truth)**
- Confirm the authoritative token file defines:
  - `primary/brand/accent` as red
  - `link` as red (or derived from primary)
  - `primary-hover`, `primary-active`, `focus-ring`, and any `primary-button-bg` token
- Confirm dark-mode equivalents exist and are updated.

**B. Propagation validation**
- Use repo-wide search to ensure old brand color values are not present outside the token source.
- In browser devtools, inspect computed styles for:
  - Primary button background uses the primary token
  - Link color uses link/primary token
  - Nav active state uses primary token

**C. Interaction-state validation (repeatable checks)**
- For **primary button** and **link**:
  - Default vs hover vs active computed colors differ (inspect `background-color`/`color`).
  - Disabled state is clearly different (opacity/color/cursor).
- For **focus**:
  - Keyboard tab to button/link/input: focus ring visible and not blending into red surfaces.

**D. Accessibility / contrast validation (explicit method)**
- Check contrast for:
  1. **Primary button text** (on-primary) vs primary background
  2. **Link text** vs page background (light and dark)
- Method:
  - Use a contrast checker (browser extension, devtools, or an online WCAG contrast tool) with the computed colors.
  - Target WCAG AA (generally 4.5:1 for normal text; 3:1 for large text).

**E. Semantic color regression validation**
- Render/visit screens with success/warning/error alerts/badges and confirm they remain semantic (not red unless “error”).
- If a component library is used, verify `success/warning/error` palette entries did not change.

---

### 5. INTEGRATION POINTS

- **Global theme application**: `src/main.tsx`, `src/App.tsx`  
  (imports of `theme.css`/`globals.css`, ThemeProvider wiring, dark-mode toggling)
- **Core tokens**: `src/styles/theme.css` and/or `src/theme.ts` / `src/theme/index.ts`
- **Tailwind**: `tailwind.config.js|ts` mapping utilities to tokens (including scales)
- **Global element styles**: `src/styles/globals.css` (`a`, `:focus-visible`, selection, etc.)
- **Key components**:
  - Buttons: `src/components/Button.tsx` (or equivalent)
  - Links: `src/components/Link.tsx` (or equivalent)
  - Nav/Header: relevant layout components
  - Forms: input components / shared form styles
  - Alerts: alert/toast components

---

### 6. EDGE CASES & CONSIDERATIONS

- **Focus ring on red buttons**: must not disappear; use offset/dual ring tokens.
- **Visited link styling**: if the app styles `:visited`, ensure it remains readable and not confusing (may be a slightly muted/darker red).
- **Disabled state clarity**: avoid “disabled but still looks clickable”; ensure contrast and cursor/opacity rules are consistent.
- **Dark mode saturation**: reds can “vibrate” on dark backgrounds; choose a red shade that remains comfortable and readable.
- **Charts/data viz**: if they use `primary`, they will become red (usually acceptable). Ensure it doesn’t break semantic meaning where color encodes categories.
- **Semantic “error” is also red**: keep error semantic tokens as-is; don’t accidentally align error exactly to brand red unless the design system intends that.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on updating centralized/core theme tokens and ensuring consumers reference them.
- DO NOT include unit tests (handled separately).
- Do not introduce new theming frameworks or heavy dependencies.
- Avoid per-component hardcoded red; use tokens/utilities/theme references.
- Preserve semantic status colors (success/warning/error) unless they were incorrectly tied to primary.
- Maintain existing codebase conventions, type safety, and backward compatibility.