### 1. CODEBASE ANALYSIS

- **Backend validation** uses **Zod** schemas under `backend/src/schemas/`. The relevant file to change is explicitly provided: `backend/src/schemas/student.schema.ts`.
- The student create/update API handlers are said to live under `backend/src/api/**`, but the exact filenames are not confirmed in the prompt. The implementation must therefore:
  1) locate the actual student create/update route handlers in `backend/src/api/` (or equivalent routing layer), and  
  2) ensure they validate request bodies using the updated `StudentSchema` (or a derived schema for updates) **before** any DB write.
- **Frontend** is Next.js App Router. The registration page is `frontend/src/app/students/new/page.tsx` and the form lives under `frontend/src/components/business/StudentRegistrationForm/`.
- Frontend validation may be custom, or may use shared utilities in `frontend/src/lib/validators.ts`. We must inspect the form implementation and integrate the trimmed-length rule in the same validation mechanism already used (to keep changes minimal and consistent).
- **Important UX/behavior requirement**: acceptance criteria #1 requires the input “does not allow more than 6 characters to be entered”. If the input is **controlled**, `maxLength` alone may not be sufficient depending on how `onChange` is implemented; we should ensure the state update path cannot result in a value longer than 6 characters being displayed/entered via typing.

---

### 2. IMPLEMENTATION STRATEGY

Implement a single business rule consistently across layers:

- **Rule**: `firstName` is required; it is **trimmed** before validation; after trimming it must be **1..6 characters** inclusive.
- **Backend**: enforce via Zod in `StudentSchema` using `z.string().trim().min(1).max(6)` with clear messages. Ensure all create/update endpoints parse with this schema (or derived schemas) and reject invalid payloads with a consistent **4xx** response **before DB writes**.
- **Frontend**: enforce in three complementary ways:
  1) **Input-level constraint**: `maxLength={6}` on the actual `<input>`.
  2) **Controlled-input guard (typing)**: ensure `onChange` does not allow the stored/displayed value to exceed 6 characters for normal typing (meeting acceptance criteria #1).
  3) **Form-level validation (paste-safe, trim-aware)**: validate `firstName.trim()` for requiredness and max length; show a friendly error; block submit while invalid.
- **Backend error surfacing**: standardize how backend validation errors are returned (status code + shape) so the frontend can reliably display them (field-level if possible).

**UX decision for paste > 6** (to remove ambiguity):
- Prefer **not silently truncating pasted values**. Allow the pasted value into state (if current form architecture allows), show an error, and block submit.  
- However, acceptance criteria #1 is about *typing*; paste behavior is covered by #2 (error + block submit).  
- If the existing input component/browser `maxLength` truncates paste automatically, we still must validate trimmed value and show required/max errors appropriately; but we should avoid adding extra truncation logic that hides the issue without feedback.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Update backend Zod schema to trim and enforce max length**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `firstName` definition to trim then validate required + max length.
- **Details**:
  - Update to: `z.string().trim().min(1, "First name is required").max(6, "First name must be at most 6 characters")`
  - Ensure `.trim()` is placed before `.min()`/`.max()` so whitespace-only becomes invalid and length is computed on trimmed value.
- **Rationale**: Backend must be the source of truth; prevents invalid records even if UI is bypassed.
- **Addresses**: Acceptance criteria #5; validation rules (trimmed, 1..6, whitespace-only invalid).

**Step 2: Identify and update backend student create handler to validate before DB write**
- **File(s)**: Locate actual handler(s) under `backend/src/api/**` responsible for **creating** students (commonly something like `backend/src/api/students/create.ts`, `backend/src/api/students/index.ts`, `backend/src/api/students/route.ts`, etc.).
- **Action**: Ensure request body is validated with the updated schema before any persistence call.
- **Details**:
  - Find the POST/create path used by the frontend registration flow.
  - Ensure it uses `StudentSchema.safeParse(body)` (or equivalent existing validation wrapper).
  - On failure:
    - Return **HTTP 422 Unprocessable Entity** (recommended for validation) *or* match existing repo convention if already standardized (if the repo uses 400, keep 400 for consistency).
    - Return a consistent error payload shape (see Step 4).
  - Confirm validation occurs **before** calling any DB create function.
- **Rationale**: Guarantees invalid `firstName` never reaches DB.
- **Addresses**: Acceptance criteria #5.

**Step 3: Identify and update backend student update handler(s) with partial schema behavior**
- **File(s)**: Locate actual handler(s) under `backend/src/api/**` responsible for **updating** students (PUT/PATCH).
- **Action**: Ensure updates validate correctly, applying the constraint when `firstName` is present.
- **Details**:
  - If updates accept partial payloads, validate with a derived schema like `StudentSchema.partial()` (or an existing UpdateStudentSchema).
  - Ensure the rule applies **only when `firstName` is provided**:
    - With `partial()`, `firstName` becomes optional; if present, it still runs `.trim().min(1).max(6)` from the base schema.
  - Reject invalid payloads with the same 4xx + error shape as create.
- **Rationale**: Keeps update semantics intact while enforcing the same business rule.
- **Addresses**: Acceptance criteria #5; avoids unintended failures when `firstName` omitted.

**Step 4: Standardize backend validation error response shape for frontend consumption**
- **File(s)**: The same API handler files from Steps 2–3 (or a shared API error utility if one exists in the repo).
- **Action**: Ensure Zod errors are returned in a predictable format.
- **Details**:
  - Prefer a shape like:
    - `{ error: "VALIDATION_ERROR", fieldErrors: { firstName: "First name must be at most 6 characters" } }`
    - or `{ issues: [{ path: ["firstName"], message: "..." }, ...] }`
  - Choose the format that best matches existing backend patterns (inspect other endpoints).
  - Ensure status code is consistent across create/update.
- **Rationale**: Frontend must “surface backend validation errors”; this is easiest with a stable error contract.
- **Addresses**: Frontend requirement to surface backend errors; acceptance criteria #5 (clear validation error response).

**Step 5: Add `maxLength={6}` to the First Name input (UI-level constraint)**
- **File(s)**:
  - `frontend/src/components/business/StudentRegistrationForm/**` (the component that renders the firstName input)
  - Potentially `frontend/src/app/students/new/page.tsx` if the input is defined there (less likely).
- **Action**: Add `maxLength={6}` to the actual `<input>` element.
- **Details**:
  - If using a custom input component, ensure the prop is forwarded to the native input (e.g., `inputProps={{ maxLength: 6 }}` or `maxLength={6}` depending on implementation).
  - Verify it ends up on the DOM node (important for RTL test later).
- **Rationale**: Meets acceptance criteria #1 for typing constraints at the browser/input level.
- **Addresses**: Acceptance criteria #1.

**Step 6: Ensure controlled input cannot exceed 6 characters via typing (state guard)**
- **File(s)**: `frontend/src/components/business/StudentRegistrationForm/**` (where `onChange` updates form state)
- **Action**: Add a guard in the `onChange` handler for firstName so typed input cannot set state to > 6 characters.
- **Details**:
  - If the form uses a generic `handleChange(field)`:
    - Add a special-case for `firstName` to ignore additional typed characters beyond 6, or to slice to 6 **only for typing**.
  - Keep paste behavior aligned with Step 7 (validation catches it). If slicing would hide paste errors, prefer:
    - For typing: enforce hard cap (meets #1).
    - For paste: allow value through if possible, then show error + block submit (meets #2).  
    - If the component cannot distinguish paste vs typing easily, rely on `maxLength` for most cases and ensure validation still runs; do not introduce complex event handling unless necessary.
- **Rationale**: `maxLength` can be insufficient in some controlled-input implementations; this ensures criterion #1 is truly met.
- **Addresses**: Acceptance criteria #1.

**Step 7: Add form-level trimmed validation (required + max 6) and block submit**
- **File(s)**:
  - `frontend/src/components/business/StudentRegistrationForm/**` (form validation logic)
  - Optionally `frontend/src/lib/validators.ts` (if validators are centralized and reused)
- **Action**: Implement/extend validation to check `firstName.trim()` length and requiredness; prevent submission when invalid.
- **Details**:
  - Validation logic:
    - `const v = firstName.trim()`
    - if `v.length === 0` → error “First name is required”
    - else if `v.length > 6` → error “First name must be at most 6 characters”
  - Ensure validation runs at least on submit, and ideally on change/blur consistent with existing UX.
  - Ensure submit handler:
    - runs validation
    - sets field error state
    - returns early (no API call) if invalid
  - Ensure submit button is disabled or submission is otherwise blocked while invalid (match existing pattern).
- **Rationale**: Catches paste cases and trimming behavior; ensures the form cannot be submitted while invalid.
- **Addresses**: Acceptance criteria #2, #3, #4.

**Step 8: Surface backend validation errors in the registration UI**
- **File(s)**:
  - `frontend/src/components/business/StudentRegistrationForm/**` (submit handler + error rendering)
  - Possibly `frontend/src/app/students/new/page.tsx` if submission is orchestrated there
  - Possibly an existing API client module (only if it exists; do not introduce new layers)
- **Action**: Map backend 4xx validation response into field/form errors.
- **Details**:
  - If backend returns `fieldErrors.firstName`, show it under the firstName field.
  - Otherwise, show a form-level error message.
  - Ensure this does not override client-side validation incorrectly; merge errors sensibly.
- **Rationale**: Even with frontend validation, backend is authoritative; user must see why it failed.
- **Addresses**: Requirement “Surface backend validation errors if returned”; supports acceptance criteria #5.

**Step 9: Confirm `/students/new` page wiring remains consistent**
- **File**: `frontend/src/app/students/new/page.tsx`
- **Action**: Verify the page uses the updated form component and that no duplicate/contradictory validation exists.
- **Details**:
  - Ensure props passed to the form (if any) don’t bypass validation.
  - Ensure navigation on success remains unchanged.
- **Rationale**: Keeps changes minimal and avoids regressions in the registration flow.
- **Addresses**: Acceptance criteria #6 (successful registration when valid).

---

### 4. VALIDATION STRATEGY

- **Backend (authoritative)**:
  - Zod schema trims then enforces `min(1)` and `max(6)`.
  - Create/update endpoints must parse with schema (or partial schema for updates) **before** DB writes.
  - On failure, return consistent **4xx** (prefer 422 unless repo uses 400) and a predictable error payload.
- **Frontend (prevent + guide)**:
  - Input-level `maxLength={6}` to prevent typing beyond 6 in the DOM.
  - Controlled-state guard to ensure typing cannot exceed 6 even in controlled inputs.
  - Form-level validation based on `trim()` to catch whitespace-only and paste/edge cases.
  - Submission blocked while invalid; errors shown inline.
  - Backend errors displayed if returned.

---

### 5. INTEGRATION POINTS

- **Backend schema**: `backend/src/schemas/student.schema.ts` is imported by student API handlers (directly or via a schema index). After updating, all consumers automatically get the new rule.
- **Backend API handlers**: actual files under `backend/src/api/**` for student create/update must be located and updated to:
  - validate using the schema
  - return consistent validation errors
- **Frontend page → form**: `frontend/src/app/students/new/page.tsx` uses `StudentRegistrationForm` components under `frontend/src/components/business/StudentRegistrationForm/`.
- **Frontend validators**: if `frontend/src/lib/validators.ts` is already used by the form, add the trimmed max-length rule there and call it from the form; otherwise implement within the form’s existing validation pattern (do not create a parallel system).

---

### 6. EDGE CASES & CONSIDERATIONS

- `" Bob "` → trimmed to `"Bob"`; valid (length 3).
- `"      "` → trimmed to `""`; invalid required.
- `"123456"` → valid; `"1234567"` → invalid.
- Exactly 6 characters after trimming is valid.
- **Update payloads**:
  - If `firstName` omitted, update should succeed (no validation error for missing field).
  - If `firstName` provided as whitespace-only, it should fail (because trimming makes it empty).
- **Controlled input behavior**:
  - Ensure typing cannot exceed 6 (acceptance #1).
  - Paste > 6 must be caught by validation and block submit (acceptance #2). If browser truncates due to `maxLength`, validation still applies to trimmed value for requiredness.
- **Error message consistency**:
  - Backend and frontend messages can differ, but both must clearly communicate the 6-character maximum.
  - Prefer aligning copy to reduce confusion.

---

### 7. IMPLEMENTATION CONSTRAINTS

- Focus ONLY on core feature implementation.
- DO NOT include unit tests in this plan (tests handled in a separate node), but ensure the implementation is testable and deterministic.
- Follow existing codebase conventions:
  - Arrow functions
  - Consolidated exports
  - Minimal changes, no new dependencies
- Maintain backward compatibility where applicable:
  - No DB migrations
  - Existing records unaffected; only new/updated submissions are constrained
- Use existing repo patterns for:
  - API routing/handlers
  - Error handling/status codes
  - Form state management and validation integration