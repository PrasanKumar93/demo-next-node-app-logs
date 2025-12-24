### 1. CODEBASE ANALYSIS
- **Frontend: locate the real Student Registration implementation (do not assume paths)**
  - Search for the registration route/page and form component(s): keywords like `StudentRegistration`, `register student`, `firstName`, `students/register`, `createStudent`.
  - Identify the **form stack** in use:
    - React Hook Form / Formik / custom state
    - Whether validation is **schema-based** (Yup/Zod/custom) or **inline rules**
  - Identify where field-level errors are rendered and what the **standard error copy style** is (single message vs multiple).
  - Confirm whether `firstName` is currently **required or optional** in the UI and in the API contract.  
    - If required today, keep it required.
    - If optional today, apply regex/max rules only when provided (non-empty after trim).

- **Backend: locate the real student registration endpoint + validation layer**
  - Find the route/controller handling student registration (e.g., `POST /students`, `POST /students/register`, etc.).
  - Identify the existing validation mechanism:
    - DTO validation (class-validator), schema validation (Joi/Zod/Yup), express middleware, or model-level validation.
  - Confirm the repo’s **standard 4xx validation status code** (commonly `400` or `422`) and the **error response shape** (e.g., `{ errors: { field: message } }`, or an array of field errors).
  - Confirm where persistence/side effects happen (service layer, events, audit logs) so validation can run **before any side effects**.

- **Model/DB constraints check**
  - Inspect the Student model/schema and DB column definition for `firstName`:
    - If there is already a length constraint, ensure it is **≤ 6**.
    - If changing it would require a migration, follow repo conventions: only add DB constraint if migrations are standard/expected for this change; otherwise enforce at request validation + model validation (if supported without migration).

---

### 2. IMPLEMENTATION STRATEGY
Implement the same `firstName` rule in **two enforcement layers** (frontend + backend) with consistent behavior and messaging:

- **Validation rules**
  - **English letters only**: `^[A-Za-z]+$` (strict ASCII letters; no spaces, punctuation, diacritics).
  - **Max length**: `6`.
  - **Required**: only if the product currently requires it (must be verified in analysis).

- **Frontend UX**
  - Must **prevent the value from ever exceeding 6 characters** (typing + paste). This is a hard cap, not just an error.
  - For **non-letter characters**, do **not** silently “fix” the input unless the app already does that for other fields. Instead:
    - Allow the user to type/paste (still capped to 6),
    - Show an inline validation error,
    - Block submit while invalid.
  - Trimming: follow existing app convention. If trimming is used elsewhere, apply it consistently (prefer trimming on blur/submit to avoid surprising cursor jumps).

- **Backend behavior**
  - Enforce the same rules in request validation (and model-level validation if that’s the established pattern).
  - Return the repo-standard **4xx** with **field-level** error details for `firstName`.
  - Ensure validation occurs **before** any DB write or side effects.

- **Error copy**
  - Use a single canonical message (or two messages if that matches existing UX pattern). Prefer one combined message for consistency:
    - **“First name must be up to 6 English letters (A–Z).”**
  - If the existing system uses separate messages (e.g., “letters only” vs “max length”), follow that pattern but keep wording consistent across FE/BE.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Confirm actual file locations and existing validation conventions**
- **File**: N/A (repo-wide search)
- **Action**: Locate the real frontend page/component, validation module, backend route/controller, and validator/schema.
- **Details**:
  - Search for `firstName` usage in student registration context.
  - Identify:
    - FE form library and where rules live
    - BE validation middleware/schema and error response format
    - Whether `firstName` is required today (FE + BE)
- **Rationale**: The “suggested files” are hypothetical; implementation must match actual repo structure.
- **Addresses**: Prevents mis-implementation; prerequisite for all AC.

---

**Step 2: Define a single canonical FE validation rule for `firstName`**
- **File**: Actual FE validation location (e.g., `frontend/validation/studentRegistration.ts` or form schema file)
- **Action**: Add/update `firstName` validation: required (if applicable), regex, max length.
- **Details**:
  - Apply `max(6)` / `maxLength: 6`
  - Apply regex `^[A-Za-z]+$`
  - Apply trimming consistent with app conventions:
    - If app trims inputs: validate against `trim(value)` and submit trimmed value.
    - If app does not trim: treat whitespace as invalid (regex already rejects).
  - Use standardized message(s) consistent with existing FE error rendering.
- **Rationale**: Centralizes FE validation and prevents drift.
- **Addresses**: AC #2, #5; validation rules.

---

**Step 3: Enforce hard cap (<= 6) at the input level for typing + paste**
- **File**: Actual Student Registration form component (page or extracted form component)
- **Action**: Ensure the input value cannot exceed 6 characters in all entry paths.
- **Details**:
  - Add `maxLength={6}` to the input element (baseline enforcement).
  - Add explicit guards to handle cases where form libs or custom handlers can still set longer values:
    - `onChange`: set value to `nextValue.slice(0, 6)` (or equivalent via form lib setter).
    - `onPaste`: intercept pasted text, compute the resulting value, and cap to 6 before setting.
  - **IME/composition note**: if the app supports composition events (common in some inputs), verify that slicing doesn’t break composition. If issues appear:
    - Defer slicing until `onCompositionEnd`, or
    - Use the existing input component’s recommended pattern for max length.
- **Rationale**: Requirement explicitly demands blocking additional input beyond 6, including paste.
- **Addresses**: AC #1.

---

**Step 4: Decide and implement the UX for non-letter characters (validate + message, not auto-strip)**
- **File**: Same FE form component + FE validation schema/rules
- **Action**: Ensure non-letters trigger inline error and block submit; do not silently remove characters unless repo convention already does.
- **Details**:
  - Keep the hard cap behavior independent: even invalid characters are still capped to 6.
  - Show inline error when regex fails.
  - Ensure submit handler is not called (or API call not made) when invalid.
- **Rationale**: Removes ambiguity: only length is “blocked”; character set is “validated with error + blocked submit.”
- **Addresses**: AC #2.

---

**Step 5: Ensure trimming behavior is consistent and does not conflict with the 6-char cap**
- **File**: FE form component and/or submit mapping layer
- **Action**: Apply trimming at the correct lifecycle point based on repo conventions.
- **Details**:
  - Preferred approach (common): trim on **blur** and/or **before submit**:
    - On blur: `setValue('firstName', value.trim().slice(0,6))`
    - On submit: ensure payload uses trimmed value
  - If trimming is not used elsewhere, do not introduce it; rely on regex to reject whitespace.
  - Ensure FE and BE both treat whitespace the same way to avoid “passes FE, fails BE” or vice versa.
- **Rationale**: Clarifies where trimming happens and how it interacts with max length.
- **Addresses**: Validation rule #5; AC #2/#5 consistency.

---

**Step 6: Add/update backend request validation for `firstName`**
- **File**: Actual backend validator/DTO/schema for student registration (e.g., `backend/validators/...`, `dto/...`, `schemas/...`)
- **Action**: Enforce `firstName` regex + max length (and required if applicable).
- **Details**:
  - Apply:
    - `max length = 6`
    - `pattern = /^[A-Za-z]+$/`
    - required vs optional based on current API contract
  - Trimming:
    - If backend already normalizes strings (trim), keep doing so; validate post-trim.
    - If backend does not trim, do not add surprising normalization unless consistent with other fields.
  - Use the same canonical error message (or the backend’s standard message format).
- **Rationale**: Backend must reject invalid payloads even if FE is bypassed.
- **Addresses**: AC #3, #4, #5.

---

**Step 7: Ensure controller/service validates before any side effects and returns standard 4xx field-level errors**
- **File**: Backend controller/route handler + any validation middleware wiring
- **Action**: Guarantee validation runs before create/update logic and before side effects (DB writes, events, logs that assume valid data).
- **Details**:
  - Ensure validator middleware executes before controller handler, or controller validates at the top and returns early.
  - **HTTP status code**: use the repo’s established convention:
    - If the app uses `422 Unprocessable Entity` for validation, use 422.
    - Otherwise use `400 Bad Request`.
  - **Error response shape**: match existing standard (field-level). Examples (choose the repo’s actual one):
    - `{ errors: { firstName: "First name must be up to 6 English letters (A–Z)." } }`
    - or `{ errors: [{ field: "firstName", message: "..." }] }`
  - Ensure no student record is created/updated on invalid input.
- **Rationale**: Meets AC and ensures consistent client handling.
- **Addresses**: AC #3, #4.

---

**Step 8: (Defense-in-depth) Add model-level constraints only if consistent and feasible**
- **File**: Backend model/schema definition (e.g., `backend/models/Student.*`, ORM schema)
- **Action**: Add/update model constraints for `firstName` where supported.
- **Details**:
  - First check current DB column length and migration policy:
    - If column is already ≤ 6: align model validation and stop.
    - If column is > 6:
      - If repo routinely includes migrations for such changes, add a migration to set length to 6.
      - If migrations are out of scope/unwanted, do **not** change DB schema; rely on request validation + model validation (if it doesn’t require migration).
  - Pattern constraints: add only if the ORM/model layer supports it and it’s consistent with existing patterns.
- **Rationale**: Prevent invalid data from other code paths while respecting migration constraints.
- **Addresses**: Strengthens AC #3/#4 across the system.

---

**Step 9: Update UI hint/help text only if the UI already uses hints**
- **File**: FE form component
- **Action**: Add a small hint like “Up to 6 letters (A–Z)” if consistent with existing design.
- **Details**:
  - Do not redesign; only add minimal helper text if the form already has helper text patterns.
- **Rationale**: Reduces user confusion without scope creep.
- **Addresses**: Supports AC #1/#2 usability.

---

**Step 10: Task documentation update (per checklist)**
- **File**: Repo’s standard documentation location (e.g., `CHANGELOG.md`, `docs/validation.md`, internal release notes)
- **Action**: Document the new `firstName` constraint and where it’s enforced (FE + BE).
- **Details**:
  - Include the exact rule: `^[A-Za-z]+$`, max 6, required status (as implemented).
  - Mention the standardized error message and status code convention.
- **Rationale**: Checklist requires task documentation.
- **Addresses**: Non-functional requirement.

---

### 4. VALIDATION STRATEGY
- **Frontend**
  - **Hard cap**: input never exceeds 6 characters (typing + paste; verify IME/composition behavior).
  - **Validation**: regex `^[A-Za-z]+$` and required (if applicable).
  - **Submission**: blocked when invalid; inline error shown using existing UI pattern.
  - **Normalization**: trim only if consistent; apply at blur/submit to avoid cursor issues.

- **Backend**
  - **Request validation**: same regex + max length (+ required if applicable).
  - **Normalization**: trim only if consistent with backend conventions.
  - **Error handling**: return repo-standard 4xx (400/422) with field-level error details.
  - **No side effects**: validation must occur before persistence/events.

---

### 5. INTEGRATION POINTS
- FE `firstName` field name must match BE payload contract (`firstName`).
- FE validation schema/rules must align with BE validator/schema to avoid drift.
- FE submit handler should map/trim/cap consistently with BE expectations.
- BE controller must use the validator middleware/schema and return errors in the format the FE already knows how to display (or at least consistent with API standards).

---

### 6. EDGE CASES & CONSIDERATIONS
- **Typing beyond 6**: must not increase value (hard cap).
- **Pasting**:
  - Paste long string → value capped to 6.
  - Paste with spaces/newlines → still capped; regex should fail and show error.
- **Non-ASCII letters** (e.g., `José`, `Łukasz`): must be rejected (strict A–Z only).
- **Punctuation** (`O'Neil`, `Anne-Marie`): rejected.
- **Empty/whitespace-only**:
  - If required: reject.
  - If optional: treat empty as allowed, but non-empty must pass regex/max after trim (if trimming is used).
- **Trimming vs cap order**:
  - If trimming is enabled, ensure consistent order (typically trim then cap then validate).
- **IME/composition events**:
  - Verify max-length enforcement doesn’t break composition; adjust to `onCompositionEnd` if needed.
- **Backend side effects**:
  - Ensure no DB write, no event publish, no audit log that assumes valid student created when validation fails.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on core feature implementation (no unit tests in this node).
- Do NOT add new dependencies; reuse existing validation tooling/patterns.
- Follow existing codebase conventions for:
  - form handling and error rendering (FE)
  - validation middleware/DTO/schema and error responses (BE)
- Ensure type safety and code quality.
- Maintain backward compatibility; only add DB migrations if that is already standard and required by existing schema practices.