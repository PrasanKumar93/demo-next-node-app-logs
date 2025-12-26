### 1. CODEBASE ANALYSIS

- **Backend validation** is handled with **Zod v3** schemas in `backend/src/schemas/`. The current `StudentSchema` defines `firstName` as `z.string().min(1, "First name is required")`, which:
  - does **not** trim outer whitespace
  - does **not** enforce a max length
  - would allow `"   "` to pass `min(1)` (because it’s length 3)
- **Frontend** uses Next.js App Router and MUI components. The student registration UI is in:
  - `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
  - likely uses a reusable `TextInput` component (`frontend/src/components/ui/TextInput/`) that should support passing `inputProps` down to MUI’s underlying input.
- **Repo conventions** to follow:
  - arrow functions only
  - consolidated exports at end of file (no inline `export const ...`)
  - tests are co-located, and **no mocks** in tests (but tests are handled in a separate node per this planning task)

### 2. IMPLEMENTATION STRATEGY

Implement the same business rule in both layers:

- **Backend (source of truth / data integrity)**: Update `StudentSchema.firstName` to:
  1. trim outer spaces before validation
  2. require non-empty after trim
  3. enforce `max(6)`
  4. provide clear error message: **`First name must be at most 6 characters`**
- **Frontend (UX prevention + validation)**:
  1. enforce typing limit via `maxLength=6`
  2. block paste that would exceed 6 characters (prevent insertion)
  3. ensure form validation trims outer spaces and blocks submit when invalid
  4. show user-friendly error messages consistent with backend

Do backend first (canonical rule), then align frontend behavior to match.

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Update backend Zod schema for trimmed + max length firstName**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `firstName` validation in `StudentSchema`.
- **Details**:
  - Replace current `firstName: z.string().min(1, "First name is required")` with a pipeline that:
    - trims outer spaces before checks (e.g., `z.string().trim()...`)
    - enforces required after trim (still use `.min(1, "First name is required")`)
    - enforces max length 6 after trim using `.max(6, "First name must be at most 6 characters")`
  - Ensure the order is: `trim()` → `min(1, ...)` → `max(6, ...)` so whitespace-only fails and length is checked on trimmed value.
- **Rationale**: Backend must reject invalid payloads consistently and prevent `"   "` and >6 char names from being stored.
- **Addresses**: Acceptance criteria #3, #4, #5; Validation rules 1–3, 6–7.

**Step 2: Ensure backend error messaging is clear and consistent**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Confirm error message copy exactly matches required spec.
- **Details**:
  - Use the exact message for max length: `First name must be at most 6 characters`
  - Keep required message as `First name is required` (already present) but ensure it triggers after trim.
- **Rationale**: Consistent error copy across UI/API reduces confusion and simplifies test assertions.
- **Addresses**: Validation rule 7.

**Step 3: Add frontend input typing constraint (maxLength=6)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Update the First Name input to enforce `maxLength`.
- **Details**:
  - Locate the First Name field (likely a `TextInput` wrapper around MUI `TextField`).
  - Pass `inputProps={{ maxLength: 6 }}` (or the project’s equivalent prop passthrough) so the underlying `<input>` has `maxLength=6`.
  - If `TextInput` does not currently forward `inputProps`, adjust usage to match existing component API (prefer changing only the form if possible; if not possible, update `TextInput` to forward `inputProps`).
- **Rationale**: Prevents users from typing beyond 6 characters (UX requirement).
- **Addresses**: Acceptance criteria #1; Validation rule 4.

**Step 4: Block over-limit paste into First Name**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Add an `onPaste` handler to First Name input.
- **Details**:
  - Implement `onPaste` such that:
    - read pasted text from clipboard (`event.clipboardData.getData("text")`)
    - compute what the resulting value length would be if paste proceeds:
      - consider current input value and selection range (so replacing selected text is handled correctly)
      - resultingLength = currentValue.length - (selectionEnd - selectionStart) + pastedText.length
    - if resultingLength > 6: call `event.preventDefault()` to block paste
  - Do **not** truncate; requirement is to block paste entirely if it would exceed.
- **Rationale**: `maxLength` alone may not fully satisfy “block paste” behavior depending on browser/component behavior; explicit prevention ensures compliance.
- **Addresses**: Acceptance criteria #2; Validation rule 5.

**Step 5: Ensure frontend validation trims outer spaces and blocks submit**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`
- **Action**: Align form validation logic with backend rule.
- **Details**:
  - Identify how the form currently validates (likely Zod v4 schema in the form or via `frontend/src/lib/validators.ts`).
  - Ensure the First Name validation:
    - trims outer spaces before required/max checks
    - fails if empty after trim
    - fails if trimmed length > 6 with message `First name must be at most 6 characters`
  - Ensure submit handler:
    - uses the trimmed value for validation (and ideally for submission payload too, so backend receives normalized data)
    - blocks submission when invalid and displays the error near the field (following existing UI patterns)
- **Rationale**: UI must not allow submission of invalid values and must match backend behavior to avoid “works in UI but fails in API”.
- **Addresses**: Acceptance criteria #3, #5; Validation rules 1–3, 7.

**Step 6: Centralize validator if repo already uses shared validators**
- **File**: `frontend/src/lib/validators.ts` (only if the form already imports from here)
- **Action**: Add or update a shared `firstName` validator.
- **Details**:
  - Create a reusable Zod schema/validator for firstName with trim + min + max(6).
  - Update `StudentRegistrationForm` to use this shared validator to avoid duplication.
- **Rationale**: Keeps rules consistent and reduces drift between components.
- **Addresses**: Validation rules 1–3, 7.

**Step 7: Ensure UI error copy matches backend**
- **File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx` (and/or `frontend/src/lib/validators.ts`)
- **Action**: Use the same error message strings as backend for max length.
- **Details**:
  - Max length message: `First name must be at most 6 characters`
  - Required message: `First name is required`
- **Rationale**: Consistency across layers; simplifies QA and automated assertions.
- **Addresses**: Validation rule 7.

### 4. VALIDATION STRATEGY

- **Backend**: Zod schema enforces:
  - `trim()` ensures outer spaces removed before checks
  - `.min(1)` ensures non-empty after trim
  - `.max(6)` ensures length <= 6 after trim
  - clear error message for max length
- **Frontend**:
  - `maxLength=6` prevents typing beyond 6
  - `onPaste` prevents paste that would exceed 6 (including selection replacement scenarios)
  - form-level validation trims and checks required/max; submission blocked when invalid
  - error messages displayed to user and aligned with backend

### 5. INTEGRATION POINTS

- **Backend**: `StudentSchema` is likely used by API handlers and/or MongoDB utility validation. Updating it will automatically enforce the rule wherever the schema is used.
- **Frontend**:
  - `StudentRegistrationForm` integrates with UI `TextInput` component; ensure `inputProps` and `onPaste` are passed through to the underlying MUI input.
  - If a shared validator exists in `frontend/src/lib/validators.ts`, integrate there to keep rules centralized.

### 6. EDGE CASES & CONSIDERATIONS

- **Whitespace-only input**: `"   "` must fail after trim (backend + frontend).
- **Leading/trailing spaces**: `"  Sam  "` should validate as `"Sam"`; ideally submit trimmed value.
- **Paste with selection**: If user selects 2 chars and pastes 4, resulting length must be computed correctly; only block if final length would exceed 6.
- **Non-ASCII characters**: Length is JS string length; requirement says “characters” with no special casing—accept as-is.
- **Consistency**: Ensure both layers use the same max length and same error copy.

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on core feature implementation (schema + UI enforcement + paste blocking + validation alignment).
- DO NOT include unit tests in this plan (handled separately).
- Follow repo conventions:
  - arrow functions only
  - consolidated exports at end of file
  - maintain type safety
- Avoid introducing new libraries; use existing Zod/MUI patterns.
- Maintain backward compatibility aside from enforcing the new validation rule going forward.