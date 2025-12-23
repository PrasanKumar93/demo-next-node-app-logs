### 1. CODEBASE ANALYSIS
After reviewing the referenced areas, the registration flow appears to be implemented end-to-end across:
- **Frontend page + form component**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` render the registration UI, including the Street Address input, required indicator (asterisk), and client-side validation wiring (likely via a shared schema).
- **Shared client validation schema**: `src/validation/registerSchema.ts` defines required/optional rules and field constraints (e.g., max length). Street Address is currently treated as required here (directly or indirectly).
- **Backend route + validation**: `server/routes/auth.ts` handles the registration endpoint and invokes a validation layer in `server/validation/register.ts` (or similar). Street Address is currently required at the API boundary.
- **Persistence layer**: `server/models/User.ts` defines the user model shape and may mark street address as required/non-nullable. The DB schema may enforce `NOT NULL` on the street address column.
- **Downstream rendering**: other UI pages may display address fields; they must handle missing street address gracefully (avoid rendering `undefined`).

Key behavioral constraint: **do not introduce new trimming/normalization** for Street Address. If whitespace-only is entered, it must be preserved as-is (client and server).

---

### 2. IMPLEMENTATION STRATEGY
Make Street Address optional consistently across all layers:
1. **Frontend UI/UX**: keep the field visible, remove required indicator and required error, update label to “Street address (optional)”, and ensure form submission does not block when empty/omitted.
2. **Frontend validation**: update schema/rules so Street Address is optional while preserving existing constraints *when provided* (e.g., max length). Avoid adding transforms like `.trim()` or preprocessors.
3. **Backend validation**: update request schema so Street Address is not required for any region/user type; keep existing constraints when present; avoid trimming/normalization.
4. **DB/model**: ensure the column allows NULL (migration if needed) and model typing reflects optional/nullable.
5. **Graceful rendering**: ensure any address display code does not show literal `undefined` or crash when street address is missing.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Locate the Street Address field definition and required UI indicator**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`
- **Action**: Identify where Street Address is rendered and how “required” is expressed (asterisk, `required` prop, aria attributes, helper text).
- **Details**:
  - Find the label text and required marker logic.
  - Find any inline error message mapping for Street Address (e.g., `errors.streetAddress?.message`).
  - Find any HTML `required` attribute or component-level `isRequired` prop.
- **Rationale**: Ensures we remove *all* required indicators and required-only error messaging.
- **Addresses**: Acceptance criteria #2

**Step 2: Update the Street Address label to clearly indicate optionality**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`
- **Action**: Change label to “Street address (optional)” (or i18n equivalent).
- **Details**:
  - If the app uses i18n, add/update the translation key rather than hardcoding.
  - Ensure optionality is conveyed in text (not only styling).
- **Rationale**: UX requirement to keep field visible but clearly optional.
- **Addresses**: Acceptance criteria #2, UX requirement

**Step 3: Remove required indicator/asterisk and required-only UI behavior**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`
- **Action**: Remove any required marker and any “Street Address is required” messaging.
- **Details**:
  - Remove asterisk rendering for this field.
  - Remove `required` attribute / `isRequired` prop for this field only.
  - Ensure blur/submit does not trigger a “required” error for empty value.
- **Rationale**: Field must not block registration.
- **Addresses**: Acceptance criteria #1, #2

**Step 4: Update client-side validation schema to make Street Address optional**
- **File**: `src/validation/registerSchema.ts`
- **Action**: Change Street Address from required to optional while preserving existing constraints when provided.
- **Details**:
  - Convert from `required()` to `optional()` / `notRequired()` / equivalent (depending on Yup/Zod/etc.).
  - Keep existing validations like `max`, regex, allowed chars, etc., but ensure they only apply when the value is present (library-specific pattern).
  - **Do not add** `.trim()`, transforms, preprocessors, or normalization.
  - Ensure schema accepts:
    - omitted field (`undefined`)
    - empty string (`""`) if the form submits it that way
    - `null` only if the system already uses nulls; otherwise prefer optional/undefined (but DB may store null).
- **Rationale**: Aligns frontend validation with new optional requirement without changing whitespace behavior.
- **Addresses**: Acceptance criteria #1, #4; Validation rules #1, #6

**Step 5: Ensure form submission payload does not force Street Address into a trimmed/normalized value**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`
- **Action**: Verify submit handler does not trim/normalize Street Address.
- **Details**:
  - If there is a `buildPayload()` or mapping step, ensure it passes the field through unchanged.
  - If the code currently trims all string fields generically, explicitly exclude Street Address from that trimming as part of this task (only if needed to meet “preserve whitespace-only”).
- **Rationale**: Must preserve whitespace-only input as entered.
- **Addresses**: Acceptance criteria #4; Validation rule #6

**Step 6: Update backend registration request validation to make Street Address optional**
- **File**: `server/validation/register.ts`
- **Action**: Relax schema so Street Address is not required.
- **Details**:
  - Change from required to optional/nullable as appropriate for the validation library.
  - Keep existing constraints when present (max length, pattern).
  - Ensure missing field does not produce 400/422.
  - **Do not add** trimming/normalization; if the validator currently trims globally, exclude Street Address from trimming (only if necessary and within scope).
- **Rationale**: Backend must accept registrations without Street Address for all users/regions.
- **Addresses**: Acceptance criteria #3, #6; Validation rules #3, #6

**Step 7: Ensure the auth registration route passes through optional Street Address without rejecting**
- **File**: `server/routes/auth.ts`
- **Action**: Confirm route handler does not enforce Street Address presence and does not transform it.
- **Details**:
  - If the route destructures required fields, ensure Street Address is not treated as required.
  - If there is a manual check like `if (!streetAddress) return error`, remove it.
  - Ensure the user creation call can accept `undefined`/`null`/`""` depending on model expectations.
- **Rationale**: Avoid hidden validation outside the schema.
- **Addresses**: Acceptance criteria #3

**Step 8: Update the User model typing/definition to allow missing/nullable Street Address**
- **File**: `server/models/User.ts`
- **Action**: Make the street address property optional/nullable in the model definition.
- **Details**:
  - Update TypeScript types/interfaces to reflect optionality.
  - If using an ORM schema/model definition, ensure it allows null.
- **Rationale**: Prevent runtime/type errors and align with DB changes.
- **Addresses**: Persistence requirements; Acceptance criteria #1, #3

**Step 9: Add/adjust DB migration to allow NULL for street address if currently NOT NULL**
- **File**: `server/migrations/*` (new migration)
- **Action**: Relax DB constraint on the street address column.
- **Details**:
  - Create a migration that alters the relevant column (e.g., `street_address`) to be nullable.
  - Ensure migration is reversible if your tooling requires down migrations.
  - Do not change default values unless necessary.
- **Rationale**: Ensure persistence works when Street Address is omitted.
- **Addresses**: Persistence requirement; Acceptance criteria #1, #3

**Step 10: Ensure any UI that displays address renders gracefully when Street Address is missing**
- **File**: Search for address rendering usage (likely profile/summary components; not listed but required by acceptance criteria)
- **Action**: Update rendering to avoid showing `undefined` or crashing.
- **Details**:
  - Use conditional rendering or fallback to `""` when street address is null/undefined.
  - Avoid concatenations that produce `"undefined, City"` etc.
- **Rationale**: Required by acceptance criteria even if not directly in registration.
- **Addresses**: Acceptance criteria #5

**Step 11: Update any shared types between client/server (if present)**
- **File**: Any shared DTO/types file (if the repo has one; otherwise skip)
- **Action**: Mark street address optional in the registration request type.
- **Details**:
  - Ensure both client and server compile with consistent expectations.
- **Rationale**: Prevent type drift and runtime mismatch.
- **Addresses**: Overall consistency; Acceptance criteria #1–#3

---

### 4. VALIDATION STRATEGY
- **Frontend**: Street Address is optional in the client schema; no required UI indicator; no required error message; submit handler passes value through unchanged (no trimming).
- **Backend**: Registration validation schema treats Street Address as optional; route handler does not add extra required checks; if provided, existing constraints still apply.
- **Persistence**: DB column nullable; model allows null/undefined; user creation works with missing/empty street address.
- **Whitespace-only**: Ensure no new `.trim()`/normalization is introduced in either client schema, submit mapping, server validation, or route handler.

---

### 5. INTEGRATION POINTS
- `Register.tsx` / `RegistrationForm.tsx` integrates with `src/validation/registerSchema.ts` via the existing form library resolver/validator.
- `server/routes/auth.ts` integrates with `server/validation/register.ts` for request validation and with `server/models/User.ts` (and ORM) for persistence.
- DB migration integrates with the ORM/migration toolchain already used in `server/migrations/*`.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Empty vs omitted**: Some form libs submit `""` for empty inputs; backend should accept both `""` and missing field (and possibly `null` if used).
- **Whitespace-only**: Must be preserved; avoid adding trimming anywhere. If there is existing global trimming, do not expand it; only adjust if it newly breaks the requirement.
- **Downstream formatting**: Any address formatting/joining must handle missing street address without producing awkward punctuation or `undefined`.
- **All regions/users**: Ensure no conditional validation re-requires Street Address based on region/user type.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus only on making Street Address optional end-to-end (UI + validation + persistence + graceful display).
- Do not add new libraries.
- Do not introduce new trimming/normalization behavior for Street Address.
- Do not include unit tests in this plan (handled by a separate node).
- Follow existing codebase conventions for i18n, validation schemas, migrations, and typing.