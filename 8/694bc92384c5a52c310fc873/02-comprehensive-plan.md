### 1. CODEBASE ANALYSIS

The feedback is: **“just focus on core theme file.”**  
So the plan is revised to **minimize branching and touching multiple layers**. The implementation will:

- Identify the **single centralized/core colors file** (the “source of truth” tokens file).
- Change **only that file** (and only the tokens that represent the brand/primary theme and its interaction states).
- Rely on existing wiring so the change **propagates globally** without editing components, Tailwind config, or provider code—unless the “core theme file” itself contains both palettes/modes.

**What to determine (read-only) before editing:**
1. **Which file is the core token source?** One of:
   - `src/styles/theme.css` (CSS variables / design tokens)
   - `src/theme.ts` or `src/theme/index.ts` (TS theme object tokens)
2. **Does that same file include dark mode tokens?**
   - CSS variables: `.dark { ... }` / `[data-theme="dark"] { ... }` / `@media (prefers-color-scheme: dark)`
   - TS theme: `lightTheme`/`darkTheme` or `createTheme({ palette: { mode } })`

**Tokens that must exist in the core file (or be added there if missing):**
- `primary` / `brand` / `accent` (base)
- `primaryHover`, `primaryActive`, `primaryDisabled` (or equivalent)
- `onPrimary` (text/icon on primary)
- `link`, `linkHover`
- `focusRing` (or ring/outline token used by focus styles)

This keeps the change **centralized** and avoids per-component hardcoding.

---

### 2. IMPLEMENTATION STRATEGY

1. **Edit only the core theme/tokens file** (the centralized source of truth).
2. Update the **primary/brand** token(s) to a **red** shade and update **related interaction tokens** (hover/active/disabled/focus/link) in the same file.
3. If the core file contains **both light and dark palettes**, update both there.
4. Do **not** modify:
   - individual components
   - Tailwind config
   - global CSS focus rules
   - ThemeProvider wiring  
   …unless those are actually part of the same “core theme file” in this repo.

This directly addresses the feedback by constraining scope to the core theme file while still meeting the acceptance criteria via token propagation.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Locate the core theme/tokens file (no code changes yet)**
- **File(s)**:  
  `src/styles/theme.css`, `src/styles/globals.css`, `src/theme.ts`, `src/theme/index.ts`
- **Action**: Determine which file is the **centralized/core color tokens** source.
- **Details**:
  - Look for definitions of primary/link/ring tokens (CSS vars or TS theme palette).
  - Confirm this file is imported/used globally (directly or via theme assembly).
- **Rationale**: Ensures we change the correct single file and avoid touching other layers.
- **Addresses**: Global propagation requirement; validation rule #1.

---

**Step 2: Update light-mode primary/brand tokens to red (in the core theme file only)**
- **File**: the identified core theme file (e.g., `src/styles/theme.css` OR `src/theme.ts`)
- **Action**: Change the primary/brand/accent token(s) to a red hue and ensure dependent tokens are consistent.
- **Details** (adapt to actual token names in the file):
  - Set `primary/brand` to a reasonable accessible red (example target: a “600–700” red range).
  - Ensure these are updated (or added) in the same file:
    - `primaryHover`: darker than primary
    - `primaryActive`: darker than hover
    - `primaryDisabled`: lighter/desaturated or intended disabled treatment
    - `onPrimary`: ensure readable (commonly `#fff`)
    - `link`: red aligned with primary (often slightly darker than button red for readability on white)
    - `linkHover`: darker than link
    - `focusRing`: visible on neutral backgrounds (often a lighter red/pink or alpha-based ring)
- **Rationale**: This is the minimum set of tokens needed to make buttons/links/focus states reflect the new brand color globally.
- **Addresses**: Acceptance criteria 1–3; validation rules #1–4.

---

**Step 3: Update dark-mode primary tokens to red (only if dark mode tokens live in the same core file)**
- **File**: same core theme file as Step 2
- **Action**: Update the dark palette’s primary/link/focus tokens to red shades appropriate for dark UI.
- **Details**:
  - If the file contains a dark block/object, update the same token set:
    - `primary`, `primaryHover`, `primaryActive`, `primaryDisabled`
    - `onPrimary`
    - `link`, `linkHover`
    - `focusRing`
  - Choose a red that remains legible on dark surfaces (often slightly brighter than light-mode primary).
- **Rationale**: Keeps branding consistent across modes while maintaining contrast.
- **Addresses**: Acceptance criterion 4; validation rules #3–5.

---

**Step 4: Ensure semantic status colors remain unchanged (within the core theme file)**
- **File**: core theme file
- **Action**: Verify success/warning/error tokens are not modified and are not accidentally derived from `primary`.
- **Details**:
  - If the core file defines semantic tokens, leave them as-is.
  - If semantic tokens reference `primary` (rare but possible), decouple them in the core file only.
- **Rationale**: Prevents regressions where alerts/badges unintentionally become red.
- **Addresses**: Acceptance criterion 5; validation rule #6.

---

**Step 5: Confirm interaction-state distinctness via token values (core file only)**
- **File**: core theme file
- **Action**: Ensure hover/active/disabled/focus tokens are visually distinct from base primary.
- **Details**:
  - `primaryHover` ≠ `primary`
  - `primaryActive` ≠ `primaryHover`
  - `primaryDisabled` clearly distinct (lighter/desaturated/opacity-based depending on system)
  - `linkHover` ≠ `link`
  - `focusRing` not identical to primary background (so it doesn’t disappear on red buttons)
- **Rationale**: Makes hover/active/disabled/focus states clearly perceivable without touching component code.
- **Addresses**: Acceptance criterion 2–3; validation rules #2–4.

---

### 4. VALIDATION STRATEGY

Validation is intentionally focused on confirming the **core theme file change propagates** (since we are not editing other files).

1. **Token-level validation (core file)**
   - Confirm primary/brand token is now red.
   - Confirm hover/active/disabled/link/focus tokens exist and are updated accordingly.
   - If dark mode exists in the same file, confirm dark tokens are updated too.

2. **Propagation validation (runtime/manual)**
   - Inspect in browser devtools computed styles:
     - Primary button background uses the primary token and is red
     - Links are red and hover is distinct
     - Focus ring is visible on keyboard focus
     - Nav active states (if tied to primary) reflect red

3. **Regression validation**
   - Check success/warning/error alerts/badges remain their semantic colors.

4. **Contrast sanity checks**
   - Primary button text vs red background is readable (AA target).
   - Link red vs page background is readable in light and dark modes.

---

### 5. INTEGRATION POINTS

Even though we won’t edit them, these are the places that must already be wired correctly for the core token change to propagate:

- Global style/theme import: `src/main.tsx` and/or `src/App.tsx`
- Core tokens file import: `src/styles/theme.css` (if CSS vars) or `src/theme.ts` (if ThemeProvider)
- Dark mode toggle mechanism (only to verify): `.dark` / `[data-theme]` / theme mode switch

---

### 6. EDGE CASES & CONSIDERATIONS

- **Focus ring on red buttons**: ensure the ring token is not the same red as the button background; use a lighter/alpha ring token in the core file.
- **Link visited state**: if the core file defines `visited` token, ensure it remains readable and not confusingly close to default link red.
- **Disabled state**: ensure disabled doesn’t look like hover/active; keep it clearly muted.
- **Error vs brand red**: error semantic red may be close to brand red; keep error tokens unchanged to avoid semantic confusion.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Edit **only the centralized/core theme tokens file** (the source of truth).
- DO NOT include unit tests (handled separately).
- Do not hardcode red in individual components.
- Preserve semantic status colors (success/warning/error).
- Maintain existing conventions and backward compatibility.