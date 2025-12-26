### 1. CODEBASE ANALYSIS

- **Backend validation** is handled with **Zod** schemas under `backend/src/schemas/`. The relevant file is `backend/src/schemas/student.schema.ts`, which currently validates `firstName` as required (e.g., `z.string().min(1, ...)`) but does **not** correctly enforce “max 7 letters” (and must not rely on `.max(7)` because that counts characters, not letters).
- **Backend error behavior** (4xx vs 5xx, and how Zod errors are surfaced) must be **verified in-repo**:
  - Identify where request payload validation happens (route handler / controller / middleware).
  - Confirm that Zod validation failures are translated into a **4xx** response and that the **message** can be made to include `First name must contain at most 7 letters`.
- **Frontend** is a Next.js App Router app:
  - `/students/new` route entry: `frontend/src/app/students/new/page.tsx`
  - Form component: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Frontend validation architecture must be confirmed**:
  - Check whether the form uses `react-hook-form`, a Zod resolver, custom validators, or a homegrown form state.
  - Check `frontend/src/lib/validators.ts` to see existing patterns (function validators vs zod schemas) and reuse them.
- **Shared contract check**:
  - Verify whether the frontend imports any shared schema/types (e.g., from a shared package, generated types, or API client validators). If so, ensure the new rule is applied consistently and doesn’t conflict.
- **Repo conventions** from `docs/coding-preferences.md` must be followed:
  - Arrow functions only
  - Consolidated exports at end of file
  - Co-located tests exist in repo (but tests are implemented in a separate node; this plan must still outline required test updates).

---

### 2. IMPLEMENTATION STRATEGY

Implement the same authoritative business rule in both layers:

1. **Single rule definition per layer (to prevent drift within that layer)**:
   - Backend: define a local helper `countUnicodeLetters()` and a constant error message string used by the Zod refinement.
   - Frontend: define a local helper and constant error message string in `frontend/src/lib/validators.ts` (or wherever validators live), and reuse it in the form.

2. **Backend enforcement (authoritative)**:
   - Update `backend/src/schemas/student.schema.ts` to enforce: `countLetters(firstName) <= 7`.
   - Use Zod `refine`/`superRefine` with Unicode letter counting via `\p{L}`.
   - Ensure the API returns a **4xx** on validation failure and includes the intended message.

3. **Frontend UX enforcement (hard-stop + inline error + submit blocking)**:
   - Implement **hard-stop** so the input value can never end up with > 7 letters:
     - Prefer **sanitizing/truncating letters-only** on paste/typing to fit the 7-letter limit while preserving non-letters (more user-friendly than fully blocking).
     - Additionally, show an inline message when the user attempts to exceed the limit (so the “hard-stop” doesn’t feel like the UI is broken).
   - Also implement **defensive validation** on submit (and optionally on blur) to block submission if invalid (in case of browser/IME edge cases).

4. **IME/composition safety**:
   - Avoid breaking IME input by not enforcing truncation mid-composition.
   - Track composition state and apply enforcement on `compositionend` (and as a fallback in `onChange` when not composing).

5. **Tests are in-scope (but not written in this node)**:
   - This plan must explicitly enumerate the backend and frontend test cases required by the task so the next node can implement them.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Confirm existing trimming/normalization behavior (backend + frontend)**
- **File**: `backend/src/schemas/student.schema.ts` (and any shared schema utilities it imports)
- **Action**: Inspect how `firstName` is currently validated/normalized.
- **Details**:
  - Determine whether the schema uses `.trim()` or preprocessors.
  - Decide and document the concrete behavior:
    - If `.trim()` is already used for names, keep it and count letters on the trimmed value.
    - If not used, do not introduce trimming silently unless consistent with existing patterns; still count letters on the raw value.
- **Rationale**: Requirements mention trimming “if already part of existing behavior”; we must be consistent to avoid unexpected behavior changes.
- **Addresses**: Validation rule #1 consistency; reduces ambiguity.

**Step 2: Add backend Unicode letter counting helper + constant error message**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Add a small helper and a message constant used by the schema.
- **Details**:
  - Implement `const countUnicodeLetters = (value: string) => (value.match(/\p{L}/gu) ?? []).length`
  - Add `const FIRST_NAME_MAX_LETTERS_ERROR = "First name must contain at most 7 letters"`
  - Keep these near the schema definition (module-local), and reuse the constant in the refinement.
- **Rationale**: Avoid duplicated regex logic and ensure consistent error copy within backend.
- **Addresses**: Backend rule enforcement; “prevent drift” within backend.

**Step 3: Enforce backend max-7-letters via Zod refinement**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Update `firstName` schema to enforce `<= 7` letters (Unicode letters only).
- **Details**:
  - Keep required validation (e.g., `.min(1, "First name is required")`).
  - Add `.refine((v) => countUnicodeLetters(v) <= 7, { message: FIRST_NAME_MAX_LETTERS_ERROR })`
  - If the schema uses `.trim()`, ensure the refinement runs on the trimmed value (i.e., apply `.trim()` before `.refine()`).
- **Rationale**: `.max(7)` is incorrect for “letters-only”; refinement is required.
- **Addresses**: Acceptance criterion #4; validation rules #2–#4.

**Step 4: Verify backend request validation returns 4xx and surfaces message**
- **File**: (repo-dependent) likely request handler/middleware files where `StudentSchema.parse()` / `safeParse()` is used
- **Action**: Confirm and, if needed, adjust error handling so invalid `firstName` yields a **4xx** with the intended message.
- **Details**:
  - Locate where student create/update endpoints validate payloads.
  - Confirm Zod errors map to 400/422 (not 500).
  - Confirm the response includes field-level errors or at least includes the message `First name must contain at most 7 letters`.
  - If the current error formatter drops refine messages, adjust formatting to include them (without changing response shape beyond existing conventions).
- **Rationale**: The task explicitly requires a 4xx validation error and clear message.
- **Addresses**: Acceptance criterion #4; validation rule #7.

**Step 5: Confirm frontend form validation architecture and where to place validator**
- **File**: 
  - `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
  - `frontend/src/lib/validators.ts`
- **Action**: Inspect how validation is currently done and decide the integration point.
- **Details**:
  - If `validators.ts` already exports field validators, add a `firstName` validator there.
  - If the form uses Zod schema locally, either:
    - add a reusable helper in `validators.ts` and use it in the Zod `refine`, or
    - add a dedicated Zod schema for `firstName` in `validators.ts` and import it.
- **Rationale**: Avoid inventing a new validation style; align with existing patterns.
- **Addresses**: Validation feedback about underspecified architecture.

**Step 6: Add frontend Unicode letter counting + validator + message constant**
- **File**: `frontend/src/lib/validators.ts`
- **Action**: Add reusable helper(s) and a validator for max 7 letters.
- **Details**:
  - Add `countUnicodeLetters()` using `/\p{L}/gu`.
  - Add `FIRST_NAME_MAX_LETTERS_ERROR = "First name must contain at most 7 letters"`.
  - Add a validator function (or zod schema) that checks:
    - required (existing required rule)
    - `countUnicodeLetters(value) <= 7`
  - Export via consolidated export block at end of file.
- **Rationale**: Centralize logic and error copy within frontend; reuse in form and tests.
- **Addresses**: Acceptance criteria #1–#3; prevents drift within FE.

**Step 7: Implement hard-stop behavior by sanitizing to max 7 letters (preserve non-letters)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Update the `firstName` input handlers so the value cannot exceed 7 letters.
- **Details** (commit to a concrete behavior):
  - Implement a helper in the component (or imported from validators if appropriate):  
    `const clampToMaxLetters = (value: string, maxLetters: number) => string`
    - Iterate through the string by Unicode codepoints (JS string iteration is codepoint-aware for most cases) and:
      - Count letters using `\p{L}` per character
      - Append characters until letter count reaches 7
      - After reaching 7 letters, still allow appending **non-letter** characters?  
        - **Decision**: Yes, allow non-letters even after 7 letters (since they don’t count).  
        - But if the user tries to add an additional letter beyond 7, that letter is dropped.
  - Wire it into input updates:
    - Maintain `firstName` as controlled value (or via form library controller).
    - On `onChange`, compute `next = clampToMaxLetters(e.target.value, 7)` and set that as the value.
    - Track `lastAttemptExceeded` (boolean) to show the inline message when clamping removed letters.
      - Example: if `next !== e.target.value`, set an error state/message.
  - **State management clarity**:
    - You do not need to “revert to previous value” if you always clamp; the clamped value becomes the new value.
    - Store the current value in the form state; store a separate UI flag for “user attempted to exceed”.
- **Rationale**: This satisfies “hard-stop” for typing and paste while being user-friendly (paste results in a best-effort value rather than nothing).
- **Addresses**: Acceptance criterion #1; validation rule #5.

**Step 8: Handle paste explicitly (optional but recommended)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Ensure paste is also hard-stopped and triggers the same message.
- **Details**:
  - If using controlled `onChange` clamping, paste is already handled.
  - Optionally add `onPaste` to:
    - read clipboard text
    - compute the would-be value
    - set clamped value
    - prevent default paste to avoid flicker
  - Keep behavior consistent with typing.
- **Rationale**: Some browsers/input setups may briefly show pasted content before React state updates; explicit paste handling can improve UX.
- **Addresses**: Acceptance criterion #1 (paste behavior).

**Step 9: IME/composition-safe enforcement**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Avoid clamping mid-composition; enforce on composition end.
- **Details**:
  - Track `isComposing` via `onCompositionStart` / `onCompositionEnd`.
  - While `isComposing` is true:
    - Allow `onChange` to update value without clamping (or clamp only non-destructively if proven safe in the codebase).
  - On `compositionend`:
    - Clamp the final value and set the “attempt exceeded” message if clamping occurred.
- **Rationale**: Prevent breaking CJK/IME input while still enforcing the rule.
- **Addresses**: Validation feedback about IME feasibility; acceptance criterion #1 robustness.

**Step 10: Block submit and show inline error when invalid (defense-in-depth)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Ensure form submission cannot proceed if `firstName` violates the rule.
- **Details**:
  - Use the validator from `frontend/src/lib/validators.ts` in the submit handler (or form library validation schema).
  - If invalid:
    - set field error state
    - render inline error message: `First name must contain at most 7 letters`
    - do not call the API
  - Ensure required validation still works (empty string rejected).
- **Rationale**: Even with hard-stop, defensive validation is required for edge cases and programmatic submission.
- **Addresses**: Acceptance criteria #2 and #3; validation rules #1–#6.

**Step 11: Confirm route wiring uses updated form**
- **File**: `frontend/src/app/students/new/page.tsx`
- **Action**: Verify the page renders the updated `StudentRegistrationForm` and no alternate path bypasses the new logic.
- **Details**:
  - Ensure no duplicate firstName input exists elsewhere.
  - Ensure props don’t override validation/handlers.
- **Rationale**: Ensures the change is active in the actual user flow.
- **Addresses**: Acceptance criteria #1–#3 end-to-end.

**Step 12: Outline required test updates (implemented in separate node)**
- **File**: 
  - `backend/src/schemas/student.schema.test.ts`
  - `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.test.tsx`
- **Action**: Add/adjust tests to cover the new rule and hard-stop behavior.
- **Details** (explicit required cases):
  - Backend Vitest:
    - accepts exactly 7 letters: `"ABCDEFG"`
    - accepts 7 letters + non-letters: `"ABC-DEFG"` (7 letters, 1 hyphen)
    - rejects 8 letters: `"ABCDEFGH"` with message containing `First name must contain at most 7 letters`
    - rejects empty string
  - Frontend Vitest + RTL:
    - typing hard-stop: attempt to type 8th letter results in value still having only 7 letters
    - paste hard-stop: paste `"ABCDEFGH"` results in clamped value with 7 letters (and message shown)
    - inline error shown when user attempts to exceed 7 letters (or when validation runs)
    - submit blocked when invalid; submit allowed when 7 letters and other required fields valid
- **Rationale**: Tests are in-scope; this plan must specify them even if implemented later.
- **Addresses**: Task scope + acceptance criteria verification.

**Step 13: Check for shared API client schemas/types and update if present**
- **File**: repo-dependent (e.g., `frontend/src/lib/api/*`, shared `packages/*`, generated types)
- **Action**: If the frontend uses a shared schema/type for student payloads, update it or ensure it remains compatible.
- **Details**:
  - Search for `StudentSchema` usage in frontend or shared packages.
  - If a shared Zod schema exists, consider reusing it (or ensure both sides implement the same rule).
- **Rationale**: Prevent mismatches where FE allows something BE rejects (or vice versa).
- **Addresses**: Validation feedback about shared contracts.

**Step 14: Ensure conventions and copy consistency**
- **File**: all modified files
- **Action**: Ensure code style and message consistency.
- **Details**:
  - Arrow functions only
  - Consolidated exports at end
  - Use the same exact error string in FE and BE (per-layer constant)
- **Rationale**: Avoid lint/style failures and copy drift.
- **Addresses**: Conventions + “keep error copy consistent”.

---

### 4. VALIDATION STRATEGY

- **Backend (authoritative)**:
  - Zod refinement counts Unicode letters via `\p{L}` and enforces `<= 7`.
  - Confirm validation failures return **4xx** and include the refine message.
- **Frontend (UX + defense-in-depth)**:
  - **Hard-stop** via clamping/sanitization ensures the input value cannot contain > 7 letters after any user action (typing/paste).
  - **Inline messaging** appears when clamping occurs and/or when validation runs.
  - **Submit blocking** re-validates and prevents API calls if invalid (covers edge cases like IME, programmatic value setting, or browser quirks).
- **Consistency**:
  - Same error copy string used across FE and BE (as constants per layer).

---

### 5. INTEGRATION POINTS

- **Backend**:
  - `backend/src/schemas/student.schema.ts` is the validation boundary for student create/update payloads.
  - Request validation middleware/controller must parse with this schema and map failures to 4xx.
- **Frontend**:
  - `StudentRegistrationForm.tsx` is the UI entry point for student creation.
  - `frontend/src/lib/validators.ts` (or existing validation module) is the shared frontend validation source.
  - `/students/new` route must render the updated form.
- **Potential shared contract**:
  - If a shared schema/types package exists, it must be checked and updated or consciously left duplicated with parity.

---

### 6. EDGE CASES & CONSIDERATIONS

- **Unicode letters**: `\p{L}` counts accented letters and non-Latin scripts as letters (desired).
- **Non-letters**: spaces, hyphens, apostrophes, punctuation, digits do not count toward 7 and should remain allowed.
- **After reaching 7 letters**:
  - Additional **letters** are dropped (hard-stop).
  - Additional **non-letters** are allowed (since they don’t increase letter count).
- **Paste**:
  - Pasting long strings should result in a clamped value (letters limited to 7) rather than rejecting the paste entirely.
- **IME/composition**:
  - Do not clamp mid-composition; clamp on `compositionend`.
- **Empty string**:
  - Still invalid due to required rule.
- **Backend/Frontend mismatch risk**:
  - Must ensure both count letters the same way (`\p{L}` with `u` flag).
- **Runtime support**:
  - Unicode property escapes require modern JS runtimes; confirm Node version and frontend build target support it (likely yes). If not, plan fallback (but only if repo constraints demand it).

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on core feature implementation.
- DO NOT write unit test code in this node (tests are implemented separately), but **do** keep the explicit test plan (Step 12) since tests are in-scope.
- Follow existing codebase conventions and patterns:
  - Arrow functions only
  - Consolidated exports at end of file
  - Keep changes minimal and consistent with existing form/validation architecture
- Ensure type safety and code quality.
- Maintain backward compatibility where applicable, except for the intended new validation rule.
- Keep error copy consistent between FE/BE: `First name must contain at most 7 letters`.