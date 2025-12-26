### 1. CODEBASE ANALYSIS

- **Backend validation** is handled with **Zod** schemas under `backend/src/schemas/`. The current `StudentSchema` in `backend/src/schemas/student.schema.ts` enforces `firstName` as `z.string().min(1, "First name is required")` but has **no max constraint**.
- **Frontend** is a Next.js App Router app. Student registration UI is expected at:
  - `frontend/src/app/students/new/page.tsx` (route entry)
  - `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx` (form component)
- There is a shared frontend validation utility file: `frontend/src/lib/validators.ts`.
- Repo conventions from `docs/coding-preferences.md`:
  - **Arrow functions only**
  - **Consolidated exports at end of file**
  - **No mocks in tests** (but tests are out of scope for this planning node)
  - File naming: lowercase with hyphens (already consistent)

Key requirement nuance: **“7 letters”** means count only Unicode letters (`\p{L}`), not total string length. Frontend must **hard-stop** typing/paste so the value can’t ever end up with >7 letters.

---

### 2. IMPLEMENTATION STRATEGY

Implement the same “max 7 Unicode letters” rule in both layers:

1. **Backend (authoritative enforcement):**
   - Add a Zod `refine`/`superRefine` on `firstName` that counts Unicode letters and rejects if count > 7.
   - Keep required validation (`min(1)`), and add a clear error message:  
     `First name must contain at most 7 letters`.

2. **Frontend (UX + prevention):**
   - Add a reusable helper/validator that:
     - counts Unicode letters
     - validates `<= 7`
     - provides the same error copy
   - Add **hard-stop input behavior**:
     - On typing/paste, compute the *next* value and if it would exceed 7 letters, prevent the update (or sanitize to the maximum allowed letters while preserving non-letters).
   - Block submit and show inline error if invalid (defense-in-depth even with hard-stop).

Keep FE/BE error copy consistent.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Add a Unicode letter counting helper (backend-local)**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Introduce a small internal helper to count letters using Unicode property escapes.
- **Details**:
  - Add something like:
    - `const countLetters = (value: string) => (value.match(/\p{L}/gu) ?? []).length;`
  - Ensure the regex uses the `u` flag and `\p{L}`.
- **Rationale**: Zod `.max(7)` checks characters, not “letters-only”.
- **Addresses**: Backend validation rule “<= 7 letters”.

**Step 2: Enforce max-7-letters in backend Zod schema**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Update `firstName` schema to include a refinement.
- **Details**:
  - Keep `z.string().min(1, "First name is required")`
  - Add `.refine((v) => countLetters(v) <= 7, { message: "First name must contain at most 7 letters" })`
  - If trimming is already used elsewhere in backend patterns, align; otherwise validate raw input as-is (but still count letters).
- **Rationale**: Prevent invalid data from being accepted/persisted.
- **Addresses**: Acceptance criterion #4; validation rules (required + <=7 letters).

**Step 3: Add/extend a reusable frontend validator for firstName**
- **File**: `frontend/src/lib/validators.ts`
- **Action**: Add a reusable “max 7 letters” validator and letter-count helper.
- **Details**:
  - Add `countLetters` using `/\p{L}/gu`.
  - Add a validator function (or zod schema if that’s the existing pattern in this file) that returns:
    - valid/invalid
    - and/or an error string: `First name must contain at most 7 letters`
  - Export from the consolidated export block at the end.
- **Rationale**: Keep logic consistent and reusable; avoid duplicating regex logic in the form.
- **Addresses**: Frontend validation rule parity with backend.

**Step 4: Implement hard-stop behavior in the Student Registration form input**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Update the `firstName` input handling to prevent >7 letters on typing and paste.
- **Details**:
  - Locate the `firstName` field wiring (likely a controlled input).
  - Implement an input handler that evaluates the *proposed next value* and blocks updates that exceed 7 letters.
  - Preferred approach (robust for typing + paste):
    - Handle `onBeforeInput` (where supported) to prevent insertion if it would exceed limit.
    - Also handle `onPaste` to prevent paste that would exceed limit.
    - Keep `onChange` as a fallback safety net: if value exceeds, revert to previous valid value or sanitize.
  - Sanitization strategy (to satisfy “hard-stop”):
    - Either **block** the event entirely when it would exceed 7 letters, leaving value unchanged, or
    - **truncate letters only** to fit 7 while preserving non-letters (more complex but user-friendly for paste).
  - Show inline error message when user attempts to exceed limit (or when validation runs).
  - Ensure submit handler checks validator and blocks submission if invalid (even though hard-stop should prevent it).
- **Rationale**: HTML `maxLength` is insufficient; requirement explicitly demands hard-stop for letters-only count.
- **Addresses**: Acceptance criteria #1 and #2; frontend validation rules.

**Step 5: Ensure form submission is blocked when invalid and error copy is consistent**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Integrate the validator into the form’s validation/submission flow.
- **Details**:
  - On submit, validate `firstName` with the shared validator.
  - If invalid, set field error state and do not call API.
  - Ensure the inline message matches backend: `First name must contain at most 7 letters`.
- **Rationale**: Defense-in-depth and meets acceptance criteria even if hard-stop fails in some browser edge cases.
- **Addresses**: Acceptance criteria #2 and #3.

**Step 6: Confirm the /students/new route uses the updated form**
- **File**: `frontend/src/app/students/new/page.tsx`
- **Action**: Verify it renders `StudentRegistrationForm` and no additional firstName constraints exist that conflict.
- **Details**:
  - If the page passes props/initial values/handlers, ensure nothing bypasses the new hard-stop logic.
- **Rationale**: Ensure the change is actually active in the user flow.
- **Addresses**: End-to-end behavior for acceptance criteria #1–#3.

**Step 7: Keep exports/types consistent with repo conventions**
- **Files**: all modified TS/TSX files
- **Action**: Ensure:
  - Arrow functions only
  - Consolidated exports at end
  - `export type` used for types
- **Rationale**: Avoid style violations and follow project standards.

---

### 4. VALIDATION STRATEGY

- **Backend**: Zod refinement counts Unicode letters via `\p{L}` and enforces `<= 7`. This guarantees API rejects invalid payloads with a 4xx validation error (as per existing error middleware behavior).
- **Frontend**:
  - **Preventative**: hard-stop input so the field value cannot become invalid (>7 letters).
  - **Defensive**: validator runs on submit (and optionally on blur/change) to show inline error and block submission if invalid.
- **Consistency**: Use the same error message string in both layers.

---

### 5. INTEGRATION POINTS

- Backend schema `StudentSchema` is the validation boundary for create/update student operations (per project structure). Updating it enforces the rule across all API consumers.
- Frontend `StudentRegistrationForm` is the UI entry point for student creation; adding hard-stop + validation ensures UX compliance.
- `frontend/src/lib/validators.ts` becomes the shared source for the letter-count rule within the frontend.

---

### 6. EDGE CASES & CONSIDERATIONS

- **Unicode letters**: `\p{L}` counts accented letters (e.g., “É”, “ñ”) as letters, which matches the requirement.
- **Non-letters**: spaces, hyphens, apostrophes, punctuation, digits should not count toward the 7-letter limit.
- **IME/composition input**: `onBeforeInput` and composition events can behave differently; keep an `onChange` fallback that enforces the rule to avoid exceeding 7 letters.
- **Paste behavior**: Decide whether to block paste entirely if it would exceed, or truncate letters-only to fit. Either satisfies “hard-stop” as long as the resulting value never exceeds 7 letters.
- **Empty string**: still invalid due to required rule (`min(1)`), both FE and BE.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on core feature implementation (no tests in this node).
- Do not change other fields (lastName, etc.).
- Follow repo conventions:
  - Arrow functions only
  - Consolidated exports at end of file
  - Type exports via `export type`
- Maintain backward compatibility except for the new validation rule (intended behavior change).
- Keep FE/BE error copy identical: `First name must contain at most 7 letters`.