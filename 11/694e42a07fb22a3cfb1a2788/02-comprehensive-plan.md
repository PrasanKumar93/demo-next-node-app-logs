### 1. CODEBASE ANALYSIS

- **Backend validation** is implemented with **Zod** schemas under `backend/src/schemas/`. The current `StudentSchema.firstName` is effectively “string with min length 1”, which:
  - does **not** trim outer whitespace before validation
  - does **not** enforce a maximum length
  - incorrectly allows whitespace-only values like `"   "` to pass `min(1)` because the raw length is > 0
- **Frontend** is a Next.js (App Router) UI using MUI inputs (directly or via a wrapper like `frontend/src/components/ui/TextInput`). The student registration form is in:
  - `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Key feasibility risk**: the form likely uses a reusable `TextInput` wrapper. We must confirm it forwards:
  - `inputProps` (to set `maxLength`)
  - `onPaste` (to block over-limit paste)
  If it does not, we must update the wrapper component to pass these through to the underlying MUI `TextField` / `<input>`.
- **Paste-blocking detail**: to correctly block paste that would exceed 6 characters, we must compute the *resulting* value length considering selection replacement. This requires access to:
  - `event.currentTarget.value`
  - `event.currentTarget.selectionStart` / `selectionEnd`
  Therefore, the `onPaste` handler must be attached to the actual `<input>` element (or an event whose `currentTarget` is the input). With MUI `TextField`, `onPaste` typically fires from the input element; with wrappers, we must ensure the handler is bound at the correct level.
- **Consistency requirement**: error copy must match across UI and API. We should define the strings once per layer (or share within the layer) to avoid drift.
- **Zod `.trim()` behavior**: Zod supports `z.string().trim()` (Zod v3+). It trims outer whitespace and returns the trimmed string for downstream checks and parsing. If the repo uses an older Zod or avoids `.trim()`, we can use `z.preprocess((v) => typeof v === "string" ? v.trim() : v, z.string()...)` as a fallback. The plan includes a verification step and fallback.

---

### 2. IMPLEMENTATION STRATEGY

Implement the business rule **consistently** in both layers:

- **Backend (source of truth / data integrity)**:
  - Trim outer spaces before validation and before producing the parsed value.
  - Require non-empty after trim.
  - Enforce max length **6** after trim.
  - Return a clear message when exceeded: **`First name must be at most 6 characters`**.
- **Frontend (UX prevention + validation)**:
  - Prevent typing beyond 6 via `maxLength=6` on the underlying `<input>`.
  - Block paste that would exceed 6 by preventing the paste event (do not truncate).
  - Validate using trimmed value; block submit when invalid and show user-friendly errors.
  - Ensure the **submitted payload uses the trimmed `firstName`**, not the raw value, to normalize data and match backend expectations.

To keep error copy consistent:
- Backend: define messages in `student.schema.ts` near the schema (or a local constants object).
- Frontend: define the same messages in the form validator module (preferably centralized in `frontend/src/lib/validators.ts` if that’s already the pattern). If the repo already has shared validator/message constants, reuse them.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Confirm current Zod version and existing schema patterns**
- **File**: `backend/package.json` (and optionally `backend/src/schemas/student.schema.ts`)
- **Action**: Verify Zod version and whether `.trim()` is already used in the codebase.
- **Details**:
  - Check if Zod is v3+ and `.trim()` is available/used.
  - If `.trim()` is not used anywhere, decide whether to introduce it or use `z.preprocess` for consistency.
- **Rationale**: Avoid introducing an API that’s incompatible with the repo’s Zod version or conventions.
- **Addresses**: Feasibility risk from validation feedback (Zod trim availability).

---

**Step 2: Update backend `StudentSchema.firstName` to trim + required + max(6)**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `firstName` validation to enforce trimming, required-ness after trim, and max length 6.
- **Details**:
  - Implement as either:
    - `z.string().trim().min(1, "First name is required").max(6, "First name must be at most 6 characters")`
    - or (fallback) `z.preprocess(...trim..., z.string().min(...).max(...))`
  - Ensure the **max length message** is exactly: `First name must be at most 6 characters`.
  - Ensure required message triggers after trim (e.g., `"   "` fails).
  - Ensure the schema output value is the trimmed string (so downstream code receives normalized data).
- **Rationale**: Backend must reject invalid payloads and normalize valid ones.
- **Addresses**: Acceptance criteria #3, #4, #5; Validation rules 1–3, 6–7.

---

**Step 3: Align backend error copy and keep it maintainable**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Centralize `firstName` validation messages in a small local constant object (if consistent with repo style).
- **Details**:
  - Example structure (conceptually): `const firstNameMessages = { required: "...", max: "..." }`
  - Use these constants in the schema definition.
- **Rationale**: Prevent message drift and simplify future changes/tests.
- **Addresses**: “Keep error copy consistent” note; Validation rule 7.

---

**Step 4: Add/adjust backend schema tests for trim + max length + required**
- **File**: `backend/src/schemas/student.schema.test.ts`
- **Action**: Add Vitest coverage for the new behavior.
- **Details** (high-level, no code):
  - Assert pass:
    - `"Samuel"` (length 6)
    - `"  Sam  "` parses to `"Sam"` (trimmed) and passes
  - Assert fail:
    - `"Samuele"` (length 7) fails with max-length message
    - `"   "` fails with required message
  - Prefer testing via `StudentSchema.safeParse(...)` and checking `success` + error issues.
- **Rationale**: Locks in backend contract and prevents regressions.
- **Addresses**: Acceptance criteria #3–#5.

---

**Step 5: Update frontend First Name input to enforce `maxLength=6`**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Ensure the underlying `<input>` has `maxLength=6`.
- **Details**:
  - If using MUI `TextField` directly: set `inputProps={{ maxLength: 6 }}`.
  - If using a wrapper `TextInput`: pass the appropriate prop supported by the wrapper (e.g., `inputProps`, `slotProps`, etc.).
- **Rationale**: Prevent typing beyond 6 characters (UX requirement).
- **Addresses**: Acceptance criteria #1; Validation rule 4.

---

**Step 6: If needed, update the shared `TextInput` wrapper to forward `inputProps` and `onPaste`**
- **File**: `frontend/src/components/ui/TextInput/*` (exact file depends on repo structure; likely `TextInput.tsx`)
- **Action**: Ensure wrapper forwards:
  - `inputProps` to the underlying MUI input
  - `onPaste` to the underlying input event handler
- **Details**:
  - Confirm the wrapper’s prop types include these props (or a generic passthrough like `...textFieldProps`).
  - Ensure `onPaste` is attached such that `event.currentTarget` is the `<input>` (or otherwise provide access to value/selection).
- **Rationale**: Without this, Step 5/7 may not actually affect the real input element, breaking acceptance criteria.
- **Addresses**: Validation feedback about feasibility risk and event targeting.

---

**Step 7: Implement paste-blocking logic for over-limit paste (no truncation)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Add an `onPaste` handler to First Name input that blocks paste if it would exceed 6 characters.
- **Details**:
  - Read pasted text: `event.clipboardData.getData("text")`
  - Compute resulting length considering selection replacement:
    - `const value = event.currentTarget.value`
    - `const start = event.currentTarget.selectionStart ?? value.length`
    - `const end = event.currentTarget.selectionEnd ?? value.length`
    - `const nextLength = value.length - (end - start) + pastedText.length`
  - If `nextLength > 6`, call `event.preventDefault()` (block paste entirely).
  - Attach handler at the input level (via wrapper forwarding or direct MUI prop) so `currentTarget` is the input.
- **Rationale**: Explicitly satisfies “block paste” requirement; `maxLength` alone may not meet the spec.
- **Addresses**: Acceptance criteria #2; Validation rule 5.

---

**Step 8: Ensure frontend validation trims outer spaces and blocks submit; submit trimmed value**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Update form validation rules and submission payload normalization for `firstName`.
- **Details**:
  - Ensure validation checks are performed on `firstName.trim()`:
    - required after trim
    - max length 6 after trim
  - Ensure the form submission is blocked when invalid and shows:
    - `First name is required` for whitespace-only
    - `First name must be at most 6 characters` for >6
  - Ensure the payload sent to the backend uses the **trimmed** `firstName` (not just validated against trimmed).
- **Rationale**: Matches backend behavior, prevents UI/API mismatch, and normalizes data.
- **Addresses**: Acceptance criteria #3, #5; Validation rules 1–3, 7.

---

**Step 9: Centralize frontend validator/messages (if repo already uses shared validators)**
- **File**: `frontend/src/lib/validators.ts` (or existing validation module used by the form)
- **Action**: Add/update a `firstName` validator with trim + required + max(6) and shared messages.
- **Details**:
  - If the form already imports validators from here, implement the rule here and consume it in the form.
  - If not, keep validation local to the form but define message constants in one place within the form file to avoid duplication.
- **Rationale**: Reduces drift and keeps error copy consistent within the frontend.
- **Addresses**: Validation feedback about message consistency and “single source of truth” within the layer.

---

**Step 10: Add/adjust frontend RTL tests for maxLength, typing limit, paste blocking, and submit behavior**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.test.tsx`
- **Action**: Add tests (no mocks) to verify UI behavior and validation.
- **Details** (high-level, no code):
  - Assert the First Name input has `maxLength=6`.
  - Typing more than 6 characters results in value length 6.
  - Pasting a string that would exceed 6 is blocked (value unchanged).
  - Whitespace-only input fails after trim; submit is blocked and error shown.
  - Exactly 6 characters allows submit when other required fields are valid.
- **Rationale**: Ensures acceptance criteria are met in the UI and prevents regressions.
- **Addresses**: Acceptance criteria #1, #2, #3, #5.

---

### 4. VALIDATION STRATEGY

- **Backend**:
  - `StudentSchema.firstName` trims outer spaces before validation and parsing.
  - Enforces:
    - required after trim (`min(1)`)
    - max length 6 after trim (`max(6)`)
  - Returns clear, stable error messages (especially for max length).
- **Frontend**:
  - Prevents invalid entry early:
    - `maxLength=6` for typing
    - `onPaste` blocks over-limit paste (including selection replacement)
  - Validates on trimmed value and blocks submission when invalid.
  - Submits normalized (trimmed) `firstName` to match backend.

---

### 5. INTEGRATION POINTS

- **Backend**:
  - Any API endpoint using `StudentSchema` (or derived create/update schemas) will automatically enforce the new rule.
  - Ensure no other schema overrides `firstName` constraints elsewhere (search for other student-related schemas).
- **Frontend**:
  - `StudentRegistrationForm` must pass `inputProps` and `onPaste` down to the actual `<input>`.
  - If `TextInput` wrapper exists, it is a critical integration point; update it if it does not forward these props.
  - If frontend uses a shared validator module, integrate the rule there to keep form logic consistent.

---

### 6. EDGE CASES & CONSIDERATIONS

- **Whitespace-only**: `"   "` must fail (after trim) in both UI and API.
- **Leading/trailing spaces**: `"  Sam  "` should validate as `"Sam"` and be submitted as `"Sam"`.
- **Paste with selection**: Replacing selected text must be accounted for when computing resulting length.
- **Paste of exactly fitting text**: If paste results in length exactly 6, allow it.
- **Non-ASCII characters**: Length is JS string length; no special casing required by spec.
- **Wrapper event targeting**: Ensure `onPaste` handler receives an event whose `currentTarget` is the input so selection/value can be read reliably.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on core feature implementation (schema + UI enforcement + paste blocking + validation alignment).
- DO NOT write actual code implementations in this plan.
- Follow existing codebase conventions:
  - arrow functions only
  - consolidated exports at end of file
  - no new third-party libraries
- Ensure type safety and code quality.
- Maintain backward compatibility where applicable (no DB migrations; only enforce going forward).