### 1. CODEBASE ANALYSIS
- **Frontend**: Locate the Student Registration page/component and identify the current form stack (e.g., React Hook Form + resolver, Formik, custom state). Confirm where validation rules live (inline rules vs shared schema under `frontend/validation/...`). Identify how errors are rendered (field-level helper text, error summary, etc.).
- **Backend**: Locate the student registration endpoint handler (controller/route) and the current validation mechanism (DTO/schema validator, middleware, model-level constraints). Identify where request payload validation is enforced and how 4xx validation errors are shaped (status code, error body format, field-level errors).
- **Shared conventions**: Check if the repo already centralizes regex/constants (e.g., `NAME_REGEX`) and standard error messages. Reuse existing patterns to avoid drift.
- **Likely relevant files (to confirm in repo)**:
  - `frontend/pages/StudentRegistration.(tsx|jsx)` and/or `frontend/components/forms/StudentRegistrationForm.(tsx|jsx)`
  - `frontend/validation/studentRegistration.(ts|js)` (or equivalent)
  - `backend/validators/studentRegistration.(ts|js)` (or equivalent)
  - `backend/controllers/students.(ts|js)` and route wiring
  - `backend/models/Student.(ts|js)` if schema constraints exist at model level

---

### 2. IMPLEMENTATION STRATEGY
Implement the same `firstName` rule in **two layers**:
1. **Frontend**: 
   - Hard-cap input to **6 characters** for both typing and paste (so the value never exceeds 6).
   - Validate **English letters only** with `^[A-Za-z]+$` and show an inline error message.
   - Block submit when invalid.
2. **Backend**:
   - Enforce the same constraints in the request validation layer (and optionally model/schema constraints if that’s the established pattern).
   - Return a **4xx** (typically 400 or 422 depending on existing conventions) with a **field-level** error for `firstName`.
   - Ensure no student record is created/updated when invalid.

Keep error copy consistent across FE/BE:  
**“First name must be up to 6 English letters (A–Z).”**

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Identify current Student Registration form + validation entry points**
- **File**: `frontend/pages/StudentRegistration.(tsx|jsx)` and/or `frontend/components/forms/StudentRegistrationForm.(tsx|jsx)`
- **Action**: Review how `firstName` is currently wired (controlled/uncontrolled), how validation is defined, and where submit is handled.
- **Details**: Determine whether validation is inline (e.g., `register('firstName', { ... })`) or via a schema (Yup/Zod/custom).
- **Rationale**: Ensures changes follow existing patterns and don’t duplicate logic.
- **Addresses**: Foundation for all acceptance criteria.

**Step 2: Add/centralize frontend validation rule for `firstName`**
- **File**: `frontend/validation/studentRegistration.(ts|js)` (or wherever current schema/rules live)
- **Action**: Add/update `firstName` validation to enforce:
  - required (if currently required)
  - `maxLength: 6`
  - regex `^[A-Za-z]+$`
  - trimming behavior consistent with existing conventions
- **Details**:
  - Use existing schema style (e.g., Yup `.matches()` + `.max(6)`, Zod `.regex()` + `.max(6)`, or custom validator).
  - Standardize message: “First name must be up to 6 English letters (A–Z).”
- **Rationale**: Single source of truth for FE validation and consistent messaging.
- **Addresses**: AC #2, #5; validation rules.

**Step 3: Enforce “never exceed 6 chars” at the input level (typing + paste)**
- **File**: `frontend/components/forms/StudentRegistrationForm.(tsx|jsx)` (or the component that renders the input)
- **Action**: Update the `firstName` input to hard-cap length.
- **Details**:
  - Add `maxLength={6}` to the input (basic browser enforcement).
  - Add an `onChange` (or equivalent) guard that:
    - trims leading/trailing whitespace if that’s the app convention (or at least on blur/submit)
    - slices to 6 characters: `value.slice(0, 6)`
    - optionally filters non-letters in-place **or** leaves them but triggers validation error (requirement says show message; it does not require auto-stripping non-letters).
  - Add an `onPaste` handler to ensure pasted content is capped to 6 (some form libs still allow longer values to be set programmatically; explicitly cap to be safe).
- **Rationale**: Requirement explicitly says block additional input beyond 6, not just show an error.
- **Addresses**: AC #1.

**Step 4: Ensure inline error rendering + submit blocking on invalid**
- **File**: `frontend/components/forms/StudentRegistrationForm.(tsx|jsx)` and/or `frontend/pages/StudentRegistration.(tsx|jsx)`
- **Action**: Ensure the form displays a field-level error for `firstName` and prevents submission when invalid.
- **Details**:
  - Confirm the form library already blocks submit when validation fails; if not, add explicit guard in submit handler.
  - Render error text near the input using existing UI components/patterns.
  - Use the standardized message for non-letters and length (can be one combined message or two messages depending on existing UX pattern; keep copy consistent).
- **Rationale**: Meets UX requirement and prevents invalid requests from the UI.
- **Addresses**: AC #2, #5.

**Step 5: Add backend request validation for `firstName` (max 6 + letters only)**
- **File**: `backend/validators/studentRegistration.(ts|js)` (or the existing request validator/DTO/schema file)
- **Action**: Update backend validation schema/rules for `firstName`.
- **Details**:
  - Enforce:
    - required (if backend currently requires it)
    - `max length = 6`
    - regex `^[A-Za-z]+$`
    - trimming if consistent with backend conventions
  - Use the same message: “First name must be up to 6 English letters (A–Z).”
- **Rationale**: Backend must reject invalid payloads even if frontend is bypassed.
- **Addresses**: AC #3, #4, #5.

**Step 6: Ensure controller/route uses validator and returns field-level 4xx**
- **File**: `backend/controllers/students.(ts|js)` (and route/middleware wiring if separate)
- **Action**: Confirm validation runs before create/update logic; ensure invalid payload returns 4xx with field error.
- **Details**:
  - If validation middleware exists, ensure it’s applied to the registration endpoint.
  - Ensure response format includes something like `{ errors: { firstName: "..." } }` or the repo’s standard structure.
  - Ensure early return on validation failure so no DB write occurs.
- **Rationale**: Acceptance criteria require 4xx + no record created/updated.
- **Addresses**: AC #3, #4.

**Step 7: (Optional but recommended if repo uses model constraints) Add model/schema constraint**
- **File**: `backend/models/Student.(ts|js)`
- **Action**: Add/update model-level constraints for `firstName` (max length 6) and possibly pattern if supported.
- **Details**:
  - If using an ORM that supports column length, set it to 6 (only if it doesn’t require a migration or if migrations are already part of normal flow).
  - If pattern constraints are supported at model level, add them; otherwise rely on request validation.
- **Rationale**: Defense-in-depth; prevents invalid data from other code paths.
- **Addresses**: Strengthens AC #3/#4 across non-registration entry points.

**Step 8: Update any UI help text / placeholders (if present)**
- **File**: Wherever the label/help text is defined (likely the form component)
- **Action**: Add subtle hint (if the UI already uses hints) like “Up to 6 letters”.
- **Details**: Keep consistent with existing design system; do not redesign.
- **Rationale**: Reduces user confusion.
- **Addresses**: Supports AC #1/#2 usability.

**Step 9: Add task documentation note (per checklist)**
- **File**: Repo’s standard changelog/release notes/task docs location (e.g., `docs/`, `CHANGELOG.md`, or internal task log)
- **Action**: Document the new constraint and error message.
- **Details**: Mention FE+BE enforcement and the exact rule.
- **Rationale**: Checklist requires task documentation.
- **Addresses**: Non-functional requirement.

---

### 4. VALIDATION STRATEGY
- **Frontend**
  - Hard cap: `maxLength={6}` + explicit cap in `onChange`/`onPaste` to guarantee value never exceeds 6.
  - Validation: regex `^[A-Za-z]+$` and required (if applicable).
  - Submission: blocked by form validation; inline error shown.
- **Backend**
  - Request validation schema enforces the same regex and max length.
  - Controller returns 4xx with field-level error and does not proceed to persistence.

---

### 5. INTEGRATION POINTS
- Frontend validation integrates with the existing form library/resolver or custom validation module already used for Student Registration.
- Backend validation integrates with existing validator middleware/DTO parsing used by the students registration endpoint.
- Error message and field name (`firstName`) must match between FE form field and BE payload contract.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Paste behavior**: Ensure pasted strings are capped to 6 even if they include newlines/spaces; apply trimming per convention.
- **Whitespace**: If trimming is standard, trim before validating; otherwise treat whitespace as invalid (since regex disallows it).
- **Empty string**: If `firstName` is required, empty should error; if optional in current system, only validate when provided (but requirement assumes required).
- **Non-ASCII letters**: Reject diacritics (e.g., “José”) per strict `A–Z` requirement.
- **Consistency**: Keep one canonical regex and one canonical message per layer to avoid mismatched UX.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on core feature implementation (no tests in this plan).
- Do NOT add new dependencies; reuse existing validation tooling.
- Follow existing codebase conventions for validation, error handling, and typing.
- Maintain backward compatibility where applicable (avoid DB migrations unless already standard/required by existing schema approach).