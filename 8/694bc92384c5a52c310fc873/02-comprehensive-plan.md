### 1. CODEBASE ANALYSIS

The provided “reference files” could not be read in this planning context (each shows `[Error reading file]`). That means the plan must be written to be robust to whichever theming mechanism the repo actually uses, while still being concrete about *where* to make changes and *how* to verify propagation.

Based on the task context and the listed likely files, the repo is expected to use one (or a combination) of these common patterns:

1. **CSS variables as core tokens** (e.g., `src/styles/theme.css` defining `--color-primary`, `--color-primary-hover`, etc.), optionally with light/dark selectors.
2. **A TS theme object** (e.g., `src/theme.ts` exporting tokens/palette used by a ThemeProvider such as MUI/Chakra/styled-components).
3. **Tailwind theme tokens** (e.g., `tailwind.config.*` mapping `primary` to CSS variables or hex values).
4. **App-level provider wiring** in `src/main.tsx` / `src/App.tsx` that applies the theme globally.

The plan below assumes the repo has a *single source of truth* for “core colors” and that other layers (Tailwind, component library, global CSS) reference those tokens. The key is: **change the centralized tokens to red and ensure all consumers still point to those tokens** (no per-component hardcoding).

---

### 2. IMPLEMENTATION STRATEGY

- **Identify the core color token source** (CSS variables file or TS theme tokens).
- **Update the primary/brand token to a red palette** (base + hover + active + subtle + focus ring), for **both light and dark modes** if present.
- **Ensure all primary UI surfaces consume the token**:
  - primary buttons
  - links
  - active nav states
  - focus rings / outlines
- **Preserve semantic colors** (success/warning/error) by not touching those tokens unless they are incorrectly derived from primary.
- **Avoid component-level hardcoding**: if any component currently hardcodes the old brand color, refactor it to use tokens.
- **Run a quick UI sweep** (locally) to confirm hover/active/disabled/focus are distinct and contrast is acceptable.

(Per instructions: no unit tests in this plan; tests are handled by a separate node.)

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Locate the “core colors” source of truth**
- **File(s)**: one of
  - `src/styles/theme.css`
  - `src/styles/globals.css`
  - `src/theme.ts`
  - `src/theme/index.ts`
- **Action**: Determine where `primary/brand/accent/link/focus` tokens are defined.
- **Details**:
  - Search for tokens like: `primary`, `brand`, `accent`, `--color-primary`, `--primary`, `link`, `focus`, `ring`, `outline`, `buttonPrimary`.
  - Confirm whether light/dark mode is implemented via:
    - `.dark` class, `[data-theme="dark"]`, `@media (prefers-color-scheme: dark)`, or separate theme objects.
- **Rationale**: Ensures we change the *central* token file so the update propagates globally.
- **Addresses**: Global propagation requirement; avoids per-component hardcoding.

---

**Step 2: Update primary/brand tokens to a red palette (light mode)**
- **File**: whichever is the core token file (most likely `src/styles/theme.css` or `src/theme.ts`)
- **Action**: Change primary/brand color tokens to red and define interaction-state shades.
- **Details** (example token set; adapt to existing naming):
  - Base primary: a reasonable accessible red (e.g., `#DC2626`-like)
  - Hover: slightly darker red
  - Active/pressed: darker still
  - Disabled: desaturated/lighter red or keep disabled neutral but ensure it’s distinct
  - Focus ring: red ring with sufficient alpha/contrast (often a lighter red glow)
  - Link color: map to primary (or a slightly darker red for readability on white)
  - If tokens exist for “primary-foreground/onPrimary”, ensure it remains readable (often white/near-white).
- **Rationale**: This is the core branding change; interaction states must remain distinct.
- **Addresses**: Acceptance criteria 1–3; validation rules 1–4.

---

**Step 3: Update primary/brand tokens to a red palette (dark mode, if supported)**
- **File**: same token source as Step 2
- **Action**: Update dark palette primary tokens to red shades appropriate for dark backgrounds.
- **Details**:
  - In dark mode, base red may need to be slightly lighter/brighter to read well on dark surfaces.
  - Ensure `onPrimary` text/icon color is still readable (often near-white).
  - Ensure focus ring is visible against dark backgrounds (may need higher luminance/alpha).
- **Rationale**: Dark mode must retain red branding without losing contrast.
- **Addresses**: Acceptance criterion 4; validation rules 3–5.

---

**Step 4: Ensure global CSS uses tokens for links, focus, and selection**
- **File**: `src/styles/globals.css` (or wherever global element styles live)
- **Action**: Confirm `a`, `:focus-visible`, and any global “ring/outline” styles reference the primary/focus tokens.
- **Details**:
  - Links: `color: var(--color-link)` or `var(--color-primary)` (depending on existing convention)
  - Focus: `outline-color` / `box-shadow` uses `--color-focus` / `--ring` token
  - Any `::selection` or highlight styles that are tied to brand should use the updated tokens
- **Rationale**: Ensures non-component HTML elements and generic focus states reflect the new theme.
- **Addresses**: Acceptance criteria 1–3; validation rules 1, 4.

---

**Step 5: Update Tailwind theme mapping (if Tailwind is used) to point to tokens**
- **File(s)**: `tailwind.config.js` and/or `tailwind.config.ts` (whichever exists/used)
- **Action**: Ensure Tailwind `colors.primary` / `colors.brand` / `ringColor` map to the centralized tokens (prefer CSS variables).
- **Details**:
  - If Tailwind currently hardcodes the old brand color, replace with `rgb(var(--...)/<alpha-value>)` pattern or `var(--color-primary)` depending on setup.
  - Ensure hover/active variants exist via tokens (e.g., `primary-hover`, `primary-active`) or via Tailwind color scale mapping.
- **Rationale**: Prevents divergence where Tailwind utilities keep the old color.
- **Addresses**: Acceptance criteria 1–3; validation rules 1–2.

---

**Step 6: Verify theme provider wiring applies updated theme globally**
- **File(s)**: `src/main.tsx`, `src/App.tsx`
- **Action**: Confirm the app wraps content in the correct ThemeProvider / CSS import so token changes apply everywhere.
- **Details**:
  - Ensure `theme.css` (or equivalent) is imported once globally.
  - If using a ThemeProvider (MUI/Chakra/etc.), ensure it consumes the updated `src/theme.ts` tokens.
- **Rationale**: Prevents “theme file updated but not applied” issues.
- **Addresses**: Acceptance criterion 1 (global), criterion 4 (dark mode).

---

**Step 7: Remove any component-level hardcoded old brand color (only if found)**
- **File(s)**: likely `src/components/Button.tsx`, `src/components/Link.tsx`, nav/header components
- **Action**: Replace hardcoded color values with references to theme tokens/utilities.
- **Details**:
  - Buttons: ensure primary variant uses `primary` tokens for bg/border and `onPrimary` for text.
  - Links: ensure link color uses `link/primary` token and hover uses `primary-hover` token.
  - Focus styles: ensure they use the global focus token (not removed).
- **Rationale**: Guarantees the red theme propagates consistently and avoids regressions.
- **Addresses**: Acceptance criteria 1–3; validation rules 1–2, 4.

---

**Step 8: Confirm semantic status colors are unchanged**
- **File(s)**: core token file(s) from Steps 2–3
- **Action**: Ensure `success/warning/error/info` tokens remain as-is and are not derived from `primary` unless intentionally designed that way.
- **Details**:
  - If the design system uses `error` as red, that’s fine—but do not overwrite success/warning.
  - If “error” previously used the old primary color incorrectly, correct it to a semantic error token (but only if clearly wrong and tied to primary).
- **Rationale**: Prevents unintended recoloring of alerts/status UI.
- **Addresses**: Acceptance criterion 5; validation rule 6.

---

**Step 9: Manual UI verification sweep (developer validation)**
- **File(s)**: none (runtime check)
- **Action**: Run the app and visually verify key surfaces.
- **Details**: Check at minimum:
  - Primary button default/hover/active/disabled
  - Links default/hover/visited (if styled)
  - Active nav item / selected tabs
  - Form inputs focus ring (keyboard `Tab` navigation)
  - Alerts: success/warning/error remain semantic
  - Dark mode (if supported): repeat checks
- **Rationale**: Confirms acceptance criteria in real UI and catches token wiring gaps.
- **Addresses**: All acceptance criteria.

---

### 4. VALIDATION STRATEGY

- **Token-level validation**: confirm the primary/brand token value is red in the centralized source (CSS variables or TS theme).
- **Propagation validation**: search for old brand hex values and ensure they are removed or only exist in token definitions.
- **Interaction-state validation**: verify hover/active/disabled are distinct by checking computed styles in devtools.
- **Accessibility validation**:
  - Ensure primary button text contrast is acceptable (white on red typically OK if red is dark enough).
  - Ensure focus ring is visible on both light and dark backgrounds.
- **Semantic color regression check**: verify success/warning/error components still use their semantic tokens.

---

### 5. INTEGRATION POINTS

- **Theme application**: `src/main.tsx` / `src/App.tsx` (ThemeProvider and/or global CSS imports).
- **Token consumption**:
  - Tailwind utilities (via `tailwind.config.*` mapping to CSS variables)
  - Component library theme (via `src/theme.ts` / `src/theme/index.ts`)
  - Global element styles (via `src/styles/globals.css`)
- **Common components**: Button, Link, Nav/Header, Form controls.

---

### 6. EDGE CASES & CONSIDERATIONS

- **Visited link color**: if styled, ensure it remains readable and consistent (may be derived from primary; consider a slightly darker/muted red).
- **Focus ring on red surfaces**: if a button is red, a red focus ring can blend in—ensure focus uses an outline/halo that contrasts (e.g., lighter red/pink ring or dual ring with neutral outer).
- **Disabled states**: avoid using the same red as enabled; ensure disabled looks clearly inactive (opacity + neutral text).
- **Dark mode contrast**: reds can appear overly saturated on dark backgrounds; choose a red that remains legible without vibrating.
- **Charts/visualizations**: if they use “primary” token, they will turn red too (usually desired). Ensure it doesn’t break meaning where color encodes categories.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on updating centralized/core theme tokens and ensuring consumers reference them.
- DO NOT add unit tests in this plan (separate node).
- Do not introduce new theming frameworks or heavy dependencies.
- Preserve semantic status colors unless explicitly tied to primary in a way that violates semantics.
- Maintain type safety and existing conventions; avoid per-component hardcoded red.