### 1. CODEBASE ANALYSIS

- **Backend validation** is handled with **Zod v3** schemas in `backend/src/schemas/`. The relevant file is:
  - `backend/src/schemas/student.schema.ts` → currently has `firstName: z.string().min(1, "First name is required")` with **no trimming** and **no max length**.
- **Backend API layer** lives under `backend/src/api/` and routes aggregate via `backend/src/routes/index.ts`. Student create/update endpoints likely parse request bodies with `StudentSchema` (directly or via a helper).
- **Frontend** is Next.js App Router. Student creation page is:
  - `frontend/src/app/students/new/page.tsx` which likely renders `StudentRegistrationForm`.
- **StudentRegistrationForm** is in:
  - `frontend/src/components/business/StudentRegistrationForm/` (form logic + validation likely here).
- **TextInput** wrapper component is in:
  - `frontend/src/components/ui/TextInput/` and should pass native input props down to MUI `TextField` (needed to enforce `maxLength` at the DOM level).
- **Validation utilities** may be centralized in:
  - `frontend/src/lib/validators.ts` (Zod v4 on frontend).
- Repo conventions:
  - Arrow functions only, no `function` keyword.
  - Exports consolidated at end of file.
  - File names are lowercase with hyphens.
  - Tests are co-located, but **tests are out of scope for this planning node**.

---

### 2. IMPLEMENTATION STRATEGY

Implement the 6-character limit as **defense-in-depth** across both layers:

1. **Backend**: Update `StudentSchema.firstName` to `trim()` then enforce:
   - required after trim (`min(1)`)
   - max length 6 after trim (`max(6, ...)`)
   This guarantees data integrity even if a client bypasses UI constraints.

2. **Frontend**:
   - Hard-prevent typing/pasting > 6 characters using native `maxLength={6}` on the underlying `<input>`.
   - Mirror backend rules in the form validation schema: trim + required + max(6).
   - Ensure submit is blocked and inline error is shown when invalid.

Implementation order: backend schema first (source of truth), then frontend input constraint + frontend validation, then ensure API error handling surfaces the backend validation message cleanly.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Enforce trimmed max length in backend StudentSchema**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `firstName` Zod rule.
- **Details**:
  - Change `firstName` from `z.string().min(1, "First name is required")`
  - To something equivalent to:
    - `z.string().trim().min(1, "First name is required").max(6, "First name must be 6 characters or fewer")`
  - Ensure trimming happens before both min/max checks (Zod’s `.trim()` does this).
- **Rationale**: Backend must reject invalid payloads and apply consistent trimming behavior.
- **Addresses**: Acceptance criteria #2, #3, #4; validation rules (trim + required + max 6).

**Step 2: Ensure create/update student endpoints use the updated schema and return clear 4xx errors**
- **File**: `backend/src/api/**` (identify the student create/update handler files used by registration)
- **Action**: Verify/adjust request validation and error response.
- **Details**:
  - Confirm the handler parses request body with `StudentSchema` (or a derived schema).
  - If parsing is not centralized, ensure both **create** and **update** paths validate with the updated schema.
  - Ensure Zod validation failures return a **4xx** (typically 400/422) with a payload that includes a field-level error for `firstName` and the message `"First name must be 6 characters or fewer"`.
  - If there is a shared error middleware, ensure it formats Zod errors consistently and includes `path`/field name.
- **Rationale**: The backend must reliably reject >6 after trim and provide actionable feedback to the frontend.
- **Addresses**: Acceptance criteria #3.

**Step 3: Add native maxLength=6 to the firstName input in the registration form**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**` (locate the component rendering the First Name field)
- **Action**: Apply native input constraint.
- **Details**:
  - On the firstName `TextInput` (or MUI `TextField`), pass `inputProps={{ maxLength: 6 }}` (or the project’s equivalent prop).
  - Ensure it applies to the actual `<input>` element (not just helper text).
- **Rationale**: UX requirement is to hard-prevent entering/pasting more than 6 characters.
- **Addresses**: Acceptance criteria #1.

**Step 4: Ensure the shared TextInput component passes inputProps through to the underlying MUI input**
- **File**: `frontend/src/components/ui/TextInput/**` (likely `TextInput.tsx`)
- **Action**: Confirm/adjust prop passthrough.
- **Details**:
  - Ensure `TextInput` accepts an `inputProps` prop (or equivalent) and forwards it to MUI `TextField`’s `inputProps`.
  - If the component currently restricts props, extend its prop types to include `inputProps?: React.InputHTMLAttributes<HTMLInputElement>` (or MUI’s expected type) and pass it through.
  - Keep export style consistent (exports at end).
- **Rationale**: Without passthrough, `maxLength` won’t be enforced at the DOM level.
- **Addresses**: Acceptance criteria #1.

**Step 5: Mirror backend validation in frontend form schema (trim + required + max 6)**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**` and/or `frontend/src/lib/validators.ts`
- **Action**: Update Zod v4 validation for firstName.
- **Details**:
  - Wherever the form schema is defined, update `firstName` to:
    - trim leading/trailing whitespace before validation
    - enforce required after trim
    - enforce max length 6 after trim with a clear message matching backend intent
  - Ensure the value that is submitted is also trimmed (either by schema transform or submit handler normalization).
- **Rationale**: Defense-in-depth; also ensures whitespace-only is caught even if user enters spaces.
- **Addresses**: Acceptance criteria #1, #2, #4.

**Step 6: Ensure submit is blocked and inline error is shown for invalid firstName**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**`
- **Action**: Wire validation result to UI state.
- **Details**:
  - Confirm the form’s submit handler checks validation and prevents API call when invalid.
  - Ensure the firstName field shows inline error text when:
    - trimmed value is empty
    - trimmed value exceeds 6 (should be rare due to maxLength, but still required)
- **Rationale**: Must prevent invalid submissions and provide clear feedback.
- **Addresses**: Acceptance criteria #1, #2.

**Step 7: Ensure /students/new page uses the updated form behavior**
- **File**: `frontend/src/app/students/new/page.tsx`
- **Action**: Verify no additional constraints needed; ensure it renders the updated form.
- **Details**:
  - Confirm the page doesn’t override props in a way that bypasses `maxLength` or validation.
  - If the page passes field configs, ensure firstName config includes maxLength.
- **Rationale**: End-to-end behavior must be present in the actual registration flow.
- **Addresses**: Acceptance criteria #1–#4 (integration).

---

### 4. VALIDATION STRATEGY

- **Backend (authoritative)**:
  - `firstName` is trimmed via Zod `.trim()`.
  - After trim: must be non-empty (`min(1)`), and length must be `<= 6` (`max(6)`).
  - Invalid payloads produce a clear 4xx validation response with a `firstName` error.

- **Frontend (UX + defense-in-depth)**:
  - Native `maxLength=6` prevents most invalid input at the source (typing/paste).
  - Form schema trims and validates again to:
    - catch whitespace-only input
    - remain consistent with backend
  - Submit handler blocks submission when invalid and shows inline errors.

---

### 5. INTEGRATION POINTS

- Backend schema change integrates wherever `StudentSchema` is used:
  - student create endpoint
  - student update endpoint
  - any DB create/update utilities that parse with schema
- Frontend `StudentRegistrationForm` integrates with:
  - `TextInput` UI component (must pass `inputProps`)
  - `/students/new` page
  - `frontend/src/lib/api.ts` submission flow (should receive already-trimmed firstName; backend still trims/validates)

---

### 6. EDGE CASES & CONSIDERATIONS

- **Whitespace-only**: `"      "` → trimmed to `""` → must fail required validation on both FE/BE.
- **Leading/trailing spaces**: `" John "` → trimmed to `"John"` → valid if length <= 6.
- **Pasting long strings**: UI should cap at 6 via `maxLength`, but still validate in schema in case of programmatic value setting.
- **Non-ASCII characters**: Zod `.max(6)` counts JS string length (UTF-16 code units). This is acceptable unless the business rule requires grapheme counting (not requested).
- **Existing records**: Out of scope to migrate/clean; only enforce for new/updated submissions.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus only on implementing the feature (no tests in this node).
- Follow repo conventions:
  - arrow functions only
  - exports consolidated at end
  - type exports as `export type`
- Keep changes minimal and backward-compatible except for the new validation rule (intended behavior change).
- Ensure trimming occurs before validation and before persistence/submission.