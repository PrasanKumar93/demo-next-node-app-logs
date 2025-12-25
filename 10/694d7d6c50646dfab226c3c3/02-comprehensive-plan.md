### 1. CODEBASE ANALYSIS

- **Backend validation** is done with **Zod** schemas in `backend/src/schemas/`. The current `StudentSchema` in `backend/src/schemas/student.schema.ts` uses `z.string().min(1, ...)` for `firstName` but **does not trim** and **does not enforce max length**.
- Backend architecture suggests API handlers live under `backend/src/api/` and should validate request bodies using schemas before DB writes (per task assumptions and folder-structure doc).
- **Frontend** is Next.js App Router. Student registration page is at `frontend/src/app/students/new/page.tsx` and uses components under `frontend/src/components/business/StudentRegistrationForm/`.
- Frontend has shared validation utilities in `frontend/src/lib/validators.ts` (likely used by the form).
- Repo conventions: **arrow functions**, **consolidated exports at end**, **kebab-case filenames**, and tests are co-located (but tests are out of scope for this planning node).

---

### 2. IMPLEMENTATION STRATEGY

Implement the same business rule in both layers:

- **Backend**: Update `StudentSchema.firstName` to `trim()` then validate `min(1)` and `max(6)`. Ensure all create/update student endpoints parse with this schema (or a derived schema) so invalid payloads are rejected before DB operations.
- **Frontend**: Add `maxLength={6}` to the first name input to prevent typing beyond 6, and add **form-level validation** that checks `firstName.trim().length <= 6` (and `>= 1`) to catch paste and whitespace-only cases. Block submit when invalid and display a clear error. If backend still returns validation errors, surface them in the UI.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Add trimmed max-length constraint to backend StudentSchema**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `firstName` Zod definition.
- **Details**:
  - Change from `z.string().min(1, "First name is required")`
  - To something like: `z.string().trim().min(1, "First name is required").max(6, "First name must be at most 6 characters")`
  - Ensure trimming happens *before* min/max checks (Zod’s `.trim()` does this).
- **Rationale**: Enforces the business rule server-side and ensures whitespace-only becomes invalid.
- **Addresses**: Backend acceptance criteria #5; validation rules (trimmed, 1..6, whitespace-only invalid).

**Step 2: Ensure student create/update API routes validate using StudentSchema**
- **File(s)**: `backend/src/api/**` (identify the student create and update handlers; likely something like `backend/src/api/students/*.ts`)
- **Action**: Confirm/adjust request validation to rely on `StudentSchema` parsing.
- **Details**:
  - In POST (create) and PUT/PATCH (update) handlers, ensure the incoming body is validated via `StudentSchema.parse(...)` or `StudentSchema.safeParse(...)`.
  - If update endpoints allow partial updates, use a derived schema (e.g., `StudentSchema.partial()`), but still ensure `firstName` uses the same trimmed max constraint (it will if derived from the updated base schema).
  - On validation failure, return a **4xx** response (commonly 400 or 422) with a clear error payload that the frontend can display.
  - Ensure validation happens **before** any DB write helper is called.
- **Rationale**: Guarantees backend enforcement even if frontend is bypassed.
- **Addresses**: Backend acceptance criteria #5.

**Step 3: Add input-level maxLength constraint in the registration UI**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**` (the specific component that renders the First Name input)
- **Action**: Add `maxLength={6}` to the First Name input component usage.
- **Details**:
  - If using a custom `TextInput` component, pass `inputProps={{ maxLength: 6 }}` (MUI pattern) or `maxLength={6}` depending on how the component is implemented.
  - Ensure the attribute ends up on the actual `<input>` element.
- **Rationale**: Meets the requirement that typing cannot exceed 6 characters.
- **Addresses**: Frontend acceptance criteria #1.

**Step 4: Add form-level trimmed validation for firstName (paste-safe)**
- **File(s)**:
  - `frontend/src/components/business/StudentRegistrationForm/**` (form state + validation logic)
  - Optionally `frontend/src/lib/validators.ts` (if validators are centralized)
- **Action**: Implement validation that checks trimmed length and requiredness.
- **Details**:
  - Add/extend a validator for firstName:
    - `const trimmed = value.trim()`
    - if `trimmed.length === 0` → “First name is required”
    - if `trimmed.length > 6` → “First name must be at most 6 characters”
  - Ensure validation runs on:
    - change/blur (depending on existing UX)
    - submit (must block submit if invalid)
  - Do **not** rely solely on `maxLength`, because paste can exceed in some controlled-input scenarios or if the component doesn’t enforce it perfectly.
- **Rationale**: Ensures consistent rule enforcement and correct trimming behavior in UI.
- **Addresses**: Frontend acceptance criteria #2, #3, #4.

**Step 5: Prevent submission while invalid and show user-friendly error**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**`
- **Action**: Wire validation state into submit button disabled state and error rendering.
- **Details**:
  - Ensure the submit handler checks validation results and returns early if invalid.
  - Display the error near the First Name field (consistent with existing form error patterns).
  - Disable submit button when the form is invalid (or at least when firstName invalid).
- **Rationale**: Explicitly satisfies “form cannot be submitted while invalid”.
- **Addresses**: Frontend acceptance criteria #2, #4.

**Step 6: Surface backend validation errors in the UI**
- **File(s)**:
  - `frontend/src/components/business/StudentRegistrationForm/**` (submit handler)
  - Possibly `frontend/src/lib/api.ts` (if it normalizes error responses)
  - `frontend/src/app/students/new/page.tsx` (if page handles submission errors)
- **Action**: Ensure backend 4xx validation errors are displayed to the user.
- **Details**:
  - When API returns validation error, map it to field error if possible (e.g., firstName) or show a form-level error message.
  - Keep behavior minimal and consistent with existing error handling (e.g., toast or inline error).
- **Rationale**: Even with frontend validation, backend is source of truth; errors must be visible.
- **Addresses**: Frontend requirement “Surface backend validation errors if returned”; supports acceptance criteria #5 indirectly.

---

### 4. VALIDATION STRATEGY

- **Backend**: Zod schema enforces `firstName` as `trimmed` then `min(1)` and `max(6)`. Any create/update route must parse with this schema before DB writes.
- **Frontend**:
  - **Input constraint**: `maxLength={6}` prevents typing beyond 6.
  - **Form validation**: uses `trim()` to validate requiredness and max length, catching paste and whitespace-only.
  - **Submission gating**: submit blocked/disabled when invalid.
  - **Backend error display**: if server rejects, user sees the message.

---

### 5. INTEGRATION POINTS

- Backend schema change integrates wherever `StudentSchema` is imported (likely via `backend/src/schemas/index.ts` or direct import).
- API handlers in `backend/src/api/` must use the updated schema (or derived partial schema for updates).
- Frontend form components under `frontend/src/components/business/StudentRegistrationForm/` integrate with the `/students/new` page.
- If `frontend/src/lib/validators.ts` is already used by the form, add the firstName rule there to keep validation consistent and reusable.

---

### 6. EDGE CASES & CONSIDERATIONS

- `" Bob "` should be treated as `"Bob"` for length checks (valid).
- `"      "` should trim to empty and fail required validation.
- `"123456"` is valid; `"1234567"` invalid.
- Ensure the UI doesn’t silently truncate pasted values without telling the user (prefer showing an error if the controlled state can exceed 6; if the browser truncates due to `maxLength`, still validate trimmed value for requiredness).
- Update endpoints: if partial updates are allowed, only validate `firstName` when provided, but still enforce max 6 when present.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus only on implementing the feature (no tests in this planning node).
- Follow repo conventions:
  - Arrow functions only
  - Consolidated exports at end of files
  - Keep changes minimal, no new dependencies
- Maintain backward compatibility aside from rejecting new/updated invalid `firstName` values.