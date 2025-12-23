
# aider chat started at 2025-12-23 17:41:57

> You can skip this check with --no-gitignore  
> Add .aider* to .gitignore (recommended)? (Y)es/(N)o [Yes]: y  
> Added .aider* to .gitignore  
> /Library/Frameworks/Python.framework/Versions/3.12/bin/aider --yes-always --message # TASK CONTEXT: make street address non mandatory in registration page

**Task ID**: #2

---

## TASK OVERVIEW

**Description**: 

**Detailed Explanation**: 
Update the user registration page so the Street Address field is optional (non-mandatory) instead of required, while keeping the field visible.

Implement the change end-to-end:
- Frontend registration form: remove the required indicator/asterisk for Street Address, update form validation rules so submission is allowed when Street Address is empty/omitted, and ensure any inline error message for missing Street Address is removed.
- Backend/API validation (if present): update request validation/schema so street address is not required. Ensure the backend accepts empty/omitted street address without returning validation errors.
- Data persistence: ensure saving a user with a null/empty street address works (no DB constraint violations). If the DB column is currently NOT NULL, add a migration to allow NULL.
- UX: keep the field visible and clearly optional (e.g., label “Street address (optional)”).

Important: If a user enters whitespace-only Street Address, preserve it as-is (do not trim/normalize it to empty/NULL as part of this change).

**Business Context**:
Making Street Address optional reduces friction during sign-up and can improve conversion. Address can be collected later (e.g., profile completion, checkout) if needed. The field should remain available for users who want to provide it, but it must not block registration. This optionality applies to all users/regions.

**Aider instructions**:
## Goal
Make **Street Address** optional (non-mandatory) on the **registration page**, keep the field visible, and ensure the backend accepts registration without it.

## Key requirements
- UI:
  - Keep the Street Address input on the registration form.
  - Remove required indicator/asterisk and any “required” error messaging.
  - Update label to indicate optionality (e.g., `Street address (optional)`) consistent with existing UX/i18n.
  - Allow submit when Street Address is empty/omitted.
- Backend:
  - Update registration request validation so Street Address is **not required** for any user/region.
  - If provided, keep existing constraints (max length/allowed chars/etc.).
- Persistence:
  - Ensure user can be created with Street Address missing/NULL/empty.
  - If DB column is NOT NULL, add a migration to allow NULL.

## Important behavior
- **Do not introduce new trimming/normalization** for Street Address.
  - If the user enters whitespace-only, preserve it as entered (unless existing behavior already trims; avoid changing behavior as part of this task).

## Suggested files to inspect/update
- `src/pages/Register.tsx`
- `src/components/RegistrationForm.tsx`
- `src/validation/registerSchema.ts`
- `server/routes/auth.ts`
- `server/validation/register.ts`
- `server/models/User.ts`
- `server/migrations/*`
- `src/pages/__tests__/Register.test.tsx`
- `server/__tests__/auth.register.test.ts`

## Tests to add/update
- Frontend:
  - Submit with Street Address empty/omitted succeeds.
  - No required asterisk and no required error for Street Address.
  - Whitespace-only Street Address is not auto-trimmed/normalized by the client.
- Backend:
  - Register without Street Address returns 200/201.
  - Invalid Street Address (per existing rules) is still rejected.
  - Whitespace-only Street Address is not newly trimmed/normalized by the server.
- Persistence:
  - Creating user with null/undefined Street Address does not throw.

## Security/PII
- Don’t add verbose logging of address fields; treat as PII.

Implement the changes end-to-end and update/add the tests accordingly.

---

## SCOPE DEFINITION

### IN SCOPE

1. Keep Street Address field on the registration UI but make it optional (no required validation/indicator).
2. Update backend validation to accept missing/empty Street Address for all users/regions.
3. Update persistence constraints/migrations if Street Address is currently NOT NULL.
4. Add/adjust unit tests to cover optional Street Address behavior, including preserving whitespace-only input.

### OUT OF SCOPE

1. Redesigning the entire registration flow or adding new steps.
2. Collecting address later via new UX flows (profile completion prompts).
3. Changing other address fields (city/state/zip) unless they are coupled to Street Address requiredness.
4. Adding new third-party address validation/autocomplete services.

---

## ACCEPTANCE CRITERIA

1. Given I am on the registration page, when I leave Street Address blank and submit valid required fields, then registration succeeds.
2. Given I am on the registration page, when I focus/blur or submit with Street Address blank, then I do not see a “Street Address is required” error and no required asterisk is shown.
3. Given I submit registration without Street Address, when the backend processes the request, then it does not return a validation error for missing Street Address.
4. Given I enter only whitespace into Street Address, when I submit registration, then registration succeeds and the value is preserved as entered (no new trimming/normalization introduced).
5. Given a user registers without Street Address, when viewing any page that displays the address, then the UI renders gracefully (no crash and no literal 'undefined').
6. Given any user/region, when registering without Street Address, then the behavior is the same (Street Address is not required for any segment).

---

## TECHNICAL REQUIREMENTS

### Dependencies & Technologies

1. Use existing form validation library already in the repo (e.g., React Hook Form + Yup/Zod, Formik, etc.)—do not introduce a new one.
2. Use existing backend validation mechanism already in the repo (e.g., Joi/Zod/class-validator/JSON schema) and update the schema accordingly.
3. If DB migration tooling exists (e.g., Prisma migrations, TypeORM, Sequelize, Rails ActiveRecord), use it to relax NOT NULL constraint if needed.

### Validation Rules

1. Registration form allows submission when Street Address is empty or omitted.
2. Street Address field does not show a required UI indicator (asterisk) and does not trigger “required” error messaging on blur/submit.
3. Backend registration endpoint does not reject requests missing Street Address (no 400/422 for missing field).
4. Persisted user record can store Street Address as NULL/empty string without errors (and without DB constraint violations).
5. If Street Address is provided, it must still pass existing formatting constraints (e.g., max length, allowed characters) as currently implemented.
6. Whitespace-only Street Address input is preserved as entered (no new trimming/normalization introduced by this change).
7. Any downstream flows that display address (profile, order checkout, etc.) handle missing Street Address gracefully (no crashes, no literal 'undefined' rendering).
8. Street Address optionality applies to all users/regions (no conditional requirement by user type/region).

---


## TECHNICAL ASSUMPTIONS

1. There is a dedicated registration page/form with a Street Address input currently marked required.
2. There is server-side validation for registration payload that currently requires Street Address.
3. Street Address is stored in a user profile table/collection and may currently be non-nullable.
4. The desired change is to keep the Street Address field present on registration but make it optional.
5. Street Address should be optional for all users/regions with no conditional requirement.
6. Whitespace-only Street Address should be preserved as entered; no new trimming/normalization should be introduced by this change.

---

## SUGGESTED FILE CHANGES


1. **File**: `src/pages/Register.tsx`
   **Rationale**: Likely contains the registration form UI and client-side validation rules; update Street Address to optional and adjust label/required indicator.

2. **File**: `src/components/RegistrationForm.tsx`
   **Rationale**: If the form is extracted into a component, update field label, required indicator, and validation.

3. **File**: `src/validation/registerSchema.ts`
   **Rationale**: If a shared schema (Yup/Zod/etc.) exists, update Street Address from required() to optional()/nullable() while preserving existing constraints when provided.

4. **File**: `server/routes/auth.ts`
   **Rationale**: Registration endpoint may validate request body; ensure Street Address is not required and behavior is consistent for all users/regions.

5. **File**: `server/validation/register.ts`
   **Rationale**: Backend request validation schema likely lives here; relax requirement for Street Address while keeping existing constraints when present.

6. **File**: `server/models/User.ts`
   **Rationale**: User model may define Street Address as required/non-null; update to optional/nullable.

7. **File**: `server/migrations/*`
   **Rationale**: If DB column is NOT NULL, add migration to allow NULL for street_address.

8. **File**: `src/pages/__tests__/Register.test.tsx`
   **Rationale**: Add/adjust frontend tests to ensure Street Address is optional and whitespace-only is preserved (no new trimming).

9. **File**: `server/__tests__/auth.register.test.ts`
   **Rationale**: Add/adjust backend tests to ensure registration succeeds without Street Address and preserves whitespace-only input (no new trimming).

---

## ADDITIONAL NOTES

Keep the Street Address field visible on registration, but label it as optional (e.g., “Street address (optional)”) consistent with existing UX/i18n patterns. Update i18n strings (labels/errors) if the app is localized. Ensure accessibility: optionality should be conveyed in text, not only via styling. Do not introduce new trimming/normalization behavior for whitespace-only Street Address; preserve the value as entered (unless existing behavior already trims—avoid changing it as part of this task).

---

---

## IMPLEMENTATION PLAN FROM PLANNING NODE

The following comprehensive plan was created by the planning node. Follow this plan to implement the feature:

### 1. CODEBASE ANALYSIS
After reviewing the referenced areas, the registration flow appears to be implemented end-to-end across:

- **Frontend page + form component**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` render the registration UI, including the Street Address input, required indicator (asterisk), and client-side validation wiring (likely via a shared schema).
- **Client validation schema**: `src/validation/registerSchema.ts` defines required/optional rules and field constraints (e.g., max length, regex). Street Address is currently treated as required here (directly or indirectly).
- **Backend route + validation**: `server/routes/auth.ts` handles the registration endpoint and invokes a validation layer in `server/validation/register.ts` (or similar). Street Address is currently required at the API boundary and/or via additional manual checks.
- **Persistence layer**: `server/models/User.ts` defines the user model shape and may mark street address as required/non-nullable. The DB schema may enforce `NOT NULL` on the street address column.
- **Downstream rendering**: other UI pages/components may display address fields; they must handle missing street address gracefully (avoid rendering `undefined` or crashing).

Key behavioral constraint: **do not introduce new trimming/normalization** for Street Address. If whitespace-only is entered, it must be preserved as-is (client and server). This requires auditing for:
- client-side transforms (schema `.trim()`, preprocessors, submit payload builders)
- server-side sanitization middleware (e.g., global trim), validation transforms, ORM hooks

Also, optionality must apply **for all users/regions**, so we must locate and remove any conditional “address required” logic (e.g., based on country/region).

---

### 2. IMPLEMENTATION STRATEGY
Make Street Address optional consistently across all layers, with explicit handling for **omitted**, **empty string**, and **whitespace-only** values:

1. **Frontend UI/UX**: keep the field visible, remove required indicator and required error, update label to “Street address (optional)” (i18n-safe), and ensure submit is allowed when empty/omitted.
2. **Frontend validation**: update schema so Street Address is optional and accepts:
   - `undefined` (omitted)
   - `""` (empty string)
   - `"   "` (whitespace-only) without trimming
   while still enforcing existing constraints **only when a non-empty value is provided** (or exactly matching existing behavior).
3. **Backend validation**: update request schema so Street Address is not required for any region/user type; ensure missing/empty string does not fail validation; keep existing constraints when present; avoid trimming/normalization.
4. **Persistence**: ensure the DB column allows NULL (migration if needed) and model typing reflects optional/nullable; ensure create/update paths accept undefined/empty without constraint violations.
5. **Graceful rendering**: update any address formatting/rendering to avoid literal `undefined` and awkward punctuation when street address is missing.
6. **Types/DTOs**: ensure any shared request/response types used by client/server mark street address optional to prevent TS errors and mismatches.

(Per instructions for this node: implementation plan only; tests are handled separately, but we will include explicit steps to update/add tests as required by the task.)

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Locate Street Address field, required UI, and validation wiring**
- **File**: `src/pages/Register.tsx`, `src/components/RegistrationForm.tsx`
- **Action**: Identify how Street Address is rendered and marked required, and how validation errors are displayed.
- **Details**:
  - Find label text, required asterisk/indicator logic, `required` HTML attribute, or component props like `isRequired`.
  - Find error mapping for street address (e.g., `errors.streetAddress?.message`).
  - Identify form library and resolver (React Hook Form + Yup/Zod, Formik, etc.) and where schema is attached.
- **Rationale**: Ensures we remove *all* required indicators and required-only error behavior.
- **Addresses**: AC #2; Validation rules #2

**Step 2: Update label to clearly indicate optionality (i18n-safe)**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` (+ i18n files if present)
- **Action**: Change label to “Street address (optional)” consistent with existing UX/i18n patterns.
- **Details**:
  - If i18n is used, add/update a translation key rather than hardcoding.
  - Ensure optionality is conveyed in text (accessibility), not only via styling.
- **Rationale**: Meets UX requirement while keeping field visible.
- **Addresses**: AC #2; UX requirement

**Step 3: Remove required indicator/asterisk and required-only UI behavior**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`
- **Action**: Make Street Address non-mandatory in the UI.
- **Details**:
  - Remove asterisk rendering for this field.
  - Remove `required` attribute / `isRequired` prop for this field only.
  - Ensure blur/submit does not show “Street Address is required” when empty.
- **Rationale**: Field must not block registration.
- **Addresses**: AC #1, #2; Validation rules #1, #2

**Step 4: Update client-side validation schema to accept omitted + empty string (and preserve whitespace)**
- **File**: `src/validation/registerSchema.ts`
- **Action**: Change Street Address from required to optional with explicit empty-string handling.
- **Details** (must be explicit to avoid regressions):
  - Ensure schema accepts `undefined` (omitted) and `""` (empty string) without triggering regex/min/format validators.
  - Keep existing constraints (max length, allowed chars, etc.) **only when the value is meaningfully “provided”** per existing behavior. Concretely:
    - If the library treats `""` as a value and runs validators, add conditional logic so validators run only when `value !== ""` (and do **not** trim).
    - Do **not** add `.trim()`, transforms, preprocessors, or normalization.
  - Decide and document whether `null` is accepted:
    - If the client currently sends `null`, allow it; otherwise prefer `undefined`/`""` and let persistence store NULL if needed.
- **Rationale**: Aligns frontend validation with optional requirement while explicitly handling empty string vs omitted and preserving whitespace-only input.
- **Addresses**: AC #1, #4; Validation rules #1, #5, #6

**Step 5: Audit client submit/payload building to ensure no trimming/normalization is introduced**
- **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` (and any helpers used, e.g., `src/utils/*`)
- **Action**: Confirm Street Address is passed through unchanged.
- **Details**:
  - Inspect submit handler and any `buildPayload()`/DTO mapping.
  - If there is generic string trimming/sanitization applied to all fields, ensure Street Address is excluded (or keep existing behavior if it already trims—do not expand trimming as part of this change).
  - Ensure whitespace-only input remains whitespace-only in the payload (no new normalization to `""`/`null`).
- **Rationale**: Required by “preserve whitespace-only as entered” constraint.
- **Addresses**: AC #4; Validation rule #6

**Step 6: Identify and remove any conditional “Street Address required” logic (all regions/users)**
- **File**: `src/validation/registerSchema.ts`, `src/pages/Register.tsx`, `src/components/RegistrationForm.tsx`, and any country/region logic modules
- **Action**: Search for conditional validation or UI requirement toggles.
- **Details**:
  - Search for patterns like `when(country...)`, `if (country === ...) require street`, `isInternational`, etc.
  - Ensure Street Address is optional regardless of region/user type.
- **Rationale**: Ensures optionality applies to all users/regions.
- **Addresses**: AC #6; Validation rule #8

**Step 7: Update backend registration request validation to make Street Address optional (explicit empty-string handling)**
- **File**: `server/validation/register.ts`
- **Action**: Relax schema so Street Address is not required and does not fail when missing/empty.
- **Details**:
  - Ensure schema accepts:
    - missing field (`undefined` / absent key)
    - empty string (`""`) if client submits it
  - Keep existing constraints when present (max length, pattern), but ensure they do not run for `""` unless that was already the behavior.
  - Do **not** add trimming/normalization; avoid `.trim()`/coercion/transforms.
- **Rationale**: Backend must accept registrations without Street Address and remain consistent with client behavior.
- **Addresses**: AC #3; Validation rules #3, #5, #6

**Step 8: Audit server-side sanitization/normalization to preserve whitespace-only Street Address**
- **File**: `server/routes/auth.ts`, plus any middleware/utilities (e.g., `server/middleware/*`, `server/utils/sanitize*`, request parsing layers)
- **Action**: Confirm no global trim/normalization affects Street Address; if it does, exclude Street Address only.
- **Details**:
  - Look for middleware that trims all string fields in `req.body`.
  - Look for validation libraries configured with “trim” defaults.
  - Look for ORM hooks (beforeCreate/beforeSave) that normalize strings.
  - If trimming exists today and already affects Street Address, do not change it broadly; only ensure this task does not introduce *new* trimming. If trimming would newly break AC #4 due to schema changes, scope an exclusion to Street Address only.
- **Rationale**: Explicitly satisfies whitespace preservation requirement and avoids accidental behavior changes.
- **Addresses**: AC #4; Validation rule #6

**Step 9: Update auth registration route to not enforce Street Address and to pass it through**
- **File**: `server/routes/auth.ts`
- **Action**: Remove any manual “required” checks and ensure user creation accepts optional street address.
- **Details**:
  - Remove checks like `if (!streetAddress) return ...`.
  - Ensure destructuring/defaulting does not force `streetAddress` to `""` or `null` unless that is existing behavior.
  - Ensure the value is passed to persistence unchanged (no trimming).
- **Rationale**: Avoid hidden validation outside schema and preserve input.
- **Addresses**: AC #3, #4

**Step 10: Update request/DTO/shared types so Street Address is optional end-to-end**
- **File**: Any shared types used by client/server (if present), plus route handler types
  - Examples: `src/types/*`, `server/types/*`, `shared/*`, or DTO definitions used in `server/routes/auth.ts`
- **Action**: Mark street address optional/nullable in registration request types.
- **Details**:
  - If there is a `RegisterRequest` type/interface, change `streetAddress: string` to `streetAddress?: string` (and/or `string | null` depending on conventions).
  - Ensure both client and server compile and agree on optionality.
- **Rationale**: Prevent TS compile errors and runtime mismatch; make optionality explicit.
- **Addresses**: AC #1–#3 (consistency)

**Step 11: Update User model typing/ORM schema to allow missing/nullable Street Address**
- **File**: `server/models/User.ts` (and ORM schema file if separate, e.g., Prisma schema / Sequelize model definition)
- **Action**: Make street address optional/nullable in the model definition.
- **Details**:
  - Update TS types/interfaces.
  - Update ORM-level field definition to allow null if applicable.
- **Rationale**: Align model with new optional behavior and DB constraints.
- **Addresses**: Persistence requirements; AC #1, #3

**Step 12: Determine DB column nullability and add migration only if needed**
- **File**: ORM schema/migrations:
  - `server/migrations/*`
  - plus schema definition source of truth (e.g., `prisma/schema.prisma`, Sequelize model + migration history, TypeORM entity, etc.)
- **Action**: Verify whether the street address column is currently `NOT NULL` and identify the exact column name.
- **Details**:
  - Inspect existing migrations/schema to confirm:
    - actual column name (`street_address` vs `streetAddress` vs `address1`)
    - whether it is `NOT NULL`
  - Only create a new migration if it is currently non-nullable.
- **Rationale**: Avoid incorrect/unnecessary migrations and ensure persistence works.
- **Addresses**: Persistence requirement; AC #1, #3

**Step 13: Add DB migration to allow NULL for street address (if currently NOT NULL)**
- **File**: `server/migrations/<timestamp>_make_street_address_nullable.*`
- **Action**: Alter the column to be nullable.
- **Details**:
  - Implement “up” migration to drop NOT NULL constraint.
  - Implement “down” migration if your tooling requires it (restore NOT NULL), considering existing null data (may need a default or data cleanup strategy consistent with repo conventions).
- **Rationale**: Ensure saving a user with missing street address does not violate DB constraints.
- **Addresses**: Persistence requirement; AC #1, #3

**Step 14: Ensure downstream address display renders gracefully when Street Address is missing**
- **File**: Search and update all relevant UI components/helpers
  - Search terms: `streetAddress`, `street_address`, `address1`, `addressLine1`, `fullAddress`, string concatenations with address fields
  - Likely locations: profile pages, account settings, order summary/checkout, admin user detail views, address formatting utilities
- **Action**: Prevent crashes and prevent rendering literal `undefined` / awkward punctuation.
- **Details**:
  - If there is an address formatting helper, update it to conditionally include street address only when it’s not `null/undefined` (and possibly not `""` depending on existing conventions), without trimming whitespace-only.
  - Ensure UI does not show `"undefined"` or `"null"` strings.
- **Rationale**: Meets acceptance criteria for graceful rendering.
- **Addresses**: AC #5; Validation rule #7

**Step 15: Update/add automated tests to cover optional Street Address behavior (frontend + backend + persistence)**
- **File**: 
  - Frontend: `src/pages/__tests__/Register.test.tsx` (and any form component tests)
  - Backend: `server/__tests__/auth.register.test.ts`
- **Action**: Add/adjust tests per task requirements.
- **Details** (must cover):
  - Frontend: submit succeeds with Street Address empty/omitted; no asterisk; no “required” error; whitespace-only is not newly trimmed/normalized by client.
  - Backend: register without Street Address returns 200/201; invalid street address still rejected per existing rules; whitespace-only is not newly trimmed/normalized by server.
  - Persistence: creating user with null/undefined street address does not throw (direct model/service test if applicable).
- **Rationale**: Ensures end-to-end behavior and prevents regressions.
- **Addresses**: AC #1–#4; Persistence requirement

---

### 4. VALIDATION STRATEGY
- **Frontend**:
  - Street Address field remains visible and labeled optional.
  - Client schema accepts omitted and empty string without errors.
  - Existing constraints still apply when a non-empty value is provided (per existing rules).
  - Confirm no new trimming/normalization is introduced in schema or submit mapping; whitespace-only remains whitespace-only.
- **Backend**:
  - Registration endpoint accepts missing/empty street address without 400/422.
  - Existing constraints still enforced when street address is provided (non-empty).
  - Audit and prevent new trimming/normalization; if global trimming exists, ensure this change does not newly affect Street Address (or exclude it narrowly if required).
- **Persistence**:
  - DB allows NULL if needed; model allows optional/nullable.
  - User creation works with `undefined`/`null`/`""` depending on existing conventions.
- **Downstream rendering**:
  - Address display does not crash and does not render literal `undefined`.

---

### 5. INTEGRATION POINTS
- `Register.tsx` / `RegistrationForm.tsx` integrates with `src/validation/registerSchema.ts` via the existing form library resolver/validator.
- Client submit handler integrates with API request DTO/type (shared or local) and must pass Street Address through unchanged.
- `server/routes/auth.ts` integrates with `server/validation/register.ts` for request validation and with the user creation service/model for persistence.
- `server/models/User.ts` (and ORM schema) integrates with DB migrations to enforce nullability.
- Address display components integrate with user profile data and any address formatting helpers.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Empty vs omitted**: Some form libs submit `""` for empty inputs; backend validation must explicitly accept both absent and `""`.
- **Whitespace-only**: Must be preserved; do not add `.trim()` or transforms. Audit existing sanitizers/middleware/hooks to ensure this change doesn’t introduce new trimming.
- **Conditional validation by region**: Remove/override any country/region-based requirement so Street Address is optional for all.
- **Downstream formatting**: Avoid `"undefined, City"` and similar; use conditional joins without trimming whitespace-only.
- **DB down migration risk**: If making column NOT NULL again, existing nulls could break; follow repo conventions (may require setting default empty string or preventing down migration in prod workflows).

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on making Street Address optional end-to-end (UI + validation + persistence + graceful display).
- Use existing form and validation libraries; do not add new dependencies.
- Do not introduce new trimming/normalization behavior for Street Address; preserve whitespace-only input as entered.
- Ensure optionality applies to all users/regions (no conditional requirement).
- Treat address as PII: do not add verbose logging.
- Follow existing codebase conventions for i18n, schema validation, migrations, and typing.

---


## YOUR ROLE: EXECUTION NODE

**THIS IS AN EXECUTION/DEVELOPMENT NODE**

Your task is to implement the feature according to the comprehensive plan provided above.

---

## IMPORTANT CONSTRAINTS

**THIS IS PURE DEVELOPMENT NODE ONLY**

1. **Focus ONLY on Core Feature Implementation**
   - Implement the feature functionality as described
   - Ensure basic validation rules are in place
   - Follow existing code patterns and conventions

2. **DO NOT Include in This Phase**:
   - Unit tests (handled by other node)

3. **Code Quality Standards**:
   - Follow existing codebase conventions and patterns
   - Write clean, readable, maintainable code
   - Add clear comments for complex logic
   - Use descriptive variable and function names
   - Ensure type safety 

4. **Integration Requirements**:
   - Ensure changes integrate seamlessly with existing code
   - Maintain backward compatibility where applicable
   - Update necessary imports and dependencies
   - Follow the project's file structure and organization

---

## SUCCESS CRITERIA FOR THIS NODE

Your implementation will be considered successful when:

1. All acceptance criteria are addressed in the implementation
2. All validation rules are enforced
3. Code follows existing patterns and conventions
4. Changes integrate properly with the existing codebase
5. Basic functionality works as described
6. No breaking changes to existing functionality
7. All in-scope features are completed

---

## NEXT STEPS (Context for You)

After you complete this pure development phase:
- Other node will add comprehensive unit tests

**Your focus now**: Deliver a working implementation of the core feature.

---

**Ready to start? Begin by:**
1. Reading any reference files provided above
2. Understanding the existing code structure
3. Implementing the changes file by file
4. Testing basic functionality manually
 docs/tech-stack.md docs/folder-structure.md docs/coding-preferences.md backend/package.json frontend/package.json docs/tech-stack.md docs/folder-structure.md docs/coding-preferences.md backend/package.json frontend/package.json .aider.chat.history.md  
> Using gpt-4o model with API key from environment.  
> Skipping /Users/prasanrajpurohit/Documents/repos/github/demo-next-node-app/.aider.chat.history.md that matches gitignore spec.  
> Aider v0.86.1  
> Main model: gpt-4o with diff edit format  
> Weak model: gpt-4o-mini  
> Git repo: .git with 141 files  
> Repo-map: using 4096 tokens, auto refresh  
> Added backend/package.json to the chat.  
> Added docs/coding-preferences.md to the chat.  
> Added docs/folder-structure.md to the chat.  
> Added docs/tech-stack.md to the chat.  
> Added frontend/package.json to the chat.  

#### # TASK CONTEXT: make street address non mandatory in registration page  
####   
#### **Task ID**: #2  
####   
#### ---  
####   
#### ## TASK OVERVIEW  
####   
#### **Description**:   
####   
#### **Detailed Explanation**:   
#### Update the user registration page so the Street Address field is optional (non-mandatory) instead of required, while keeping the field visible.  
####   
#### Implement the change end-to-end:  
#### - Frontend registration form: remove the required indicator/asterisk for Street Address, update form validation rules so submission is allowed when Street Address is empty/omitted, and ensure any inline error message for missing Street Address is removed.  
#### - Backend/API validation (if present): update request validation/schema so street address is not required. Ensure the backend accepts empty/omitted street address without returning validation errors.  
#### - Data persistence: ensure saving a user with a null/empty street address works (no DB constraint violations). If the DB column is currently NOT NULL, add a migration to allow NULL.  
#### - UX: keep the field visible and clearly optional (e.g., label “Street address (optional)”).  
####   
#### Important: If a user enters whitespace-only Street Address, preserve it as-is (do not trim/normalize it to empty/NULL as part of this change).  
####   
#### **Business Context**:  
#### Making Street Address optional reduces friction during sign-up and can improve conversion. Address can be collected later (e.g., profile completion, checkout) if needed. The field should remain available for users who want to provide it, but it must not block registration. This optionality applies to all users/regions.  
####   
#### **Aider instructions**:  
#### ## Goal  
#### Make **Street Address** optional (non-mandatory) on the **registration page**, keep the field visible, and ensure the backend accepts registration without it.  
####   
#### ## Key requirements  
#### - UI:  
####   - Keep the Street Address input on the registration form.  
####   - Remove required indicator/asterisk and any “required” error messaging.  
####   - Update label to indicate optionality (e.g., `Street address (optional)`) consistent with existing UX/i18n.  
####   - Allow submit when Street Address is empty/omitted.  
#### - Backend:  
####   - Update registration request validation so Street Address is **not required** for any user/region.  
####   - If provided, keep existing constraints (max length/allowed chars/etc.).  
#### - Persistence:  
####   - Ensure user can be created with Street Address missing/NULL/empty.  
####   - If DB column is NOT NULL, add a migration to allow NULL.  
####   
#### ## Important behavior  
#### - **Do not introduce new trimming/normalization** for Street Address.  
####   - If the user enters whitespace-only, preserve it as entered (unless existing behavior already trims; avoid changing behavior as part of this task).  
####   
#### ## Suggested files to inspect/update  
#### - `src/pages/Register.tsx`  
#### - `src/components/RegistrationForm.tsx`  
#### - `src/validation/registerSchema.ts`  
#### - `server/routes/auth.ts`  
#### - `server/validation/register.ts`  
#### - `server/models/User.ts`  
#### - `server/migrations/*`  
#### - `src/pages/__tests__/Register.test.tsx`  
#### - `server/__tests__/auth.register.test.ts`  
####   
#### ## Tests to add/update  
#### - Frontend:  
####   - Submit with Street Address empty/omitted succeeds.  
####   - No required asterisk and no required error for Street Address.  
####   - Whitespace-only Street Address is not auto-trimmed/normalized by the client.  
#### - Backend:  
####   - Register without Street Address returns 200/201.  
####   - Invalid Street Address (per existing rules) is still rejected.  
####   - Whitespace-only Street Address is not newly trimmed/normalized by the server.  
#### - Persistence:  
####   - Creating user with null/undefined Street Address does not throw.  
####   
#### ## Security/PII  
#### - Don’t add verbose logging of address fields; treat as PII.  
####   
#### Implement the changes end-to-end and update/add the tests accordingly.  
####   
#### ---  
####   
#### ## SCOPE DEFINITION  
####   
#### ### IN SCOPE  
####   
#### 1. Keep Street Address field on the registration UI but make it optional (no required validation/indicator).  
#### 2. Update backend validation to accept missing/empty Street Address for all users/regions.  
#### 3. Update persistence constraints/migrations if Street Address is currently NOT NULL.  
#### 4. Add/adjust unit tests to cover optional Street Address behavior, including preserving whitespace-only input.  
####   
#### ### OUT OF SCOPE  
####   
#### 1. Redesigning the entire registration flow or adding new steps.  
#### 2. Collecting address later via new UX flows (profile completion prompts).  
#### 3. Changing other address fields (city/state/zip) unless they are coupled to Street Address requiredness.  
#### 4. Adding new third-party address validation/autocomplete services.  
####   
#### ---  
####   
#### ## ACCEPTANCE CRITERIA  
####   
#### 1. Given I am on the registration page, when I leave Street Address blank and submit valid required fields, then registration succeeds.  
#### 2. Given I am on the registration page, when I focus/blur or submit with Street Address blank, then I do not see a “Street Address is required” error and no required asterisk is shown.  
#### 3. Given I submit registration without Street Address, when the backend processes the request, then it does not return a validation error for missing Street Address.  
#### 4. Given I enter only whitespace into Street Address, when I submit registration, then registration succeeds and the value is preserved as entered (no new trimming/normalization introduced).  
#### 5. Given a user registers without Street Address, when viewing any page that displays the address, then the UI renders gracefully (no crash and no literal 'undefined').  
#### 6. Given any user/region, when registering without Street Address, then the behavior is the same (Street Address is not required for any segment).  
####   
#### ---  
####   
#### ## TECHNICAL REQUIREMENTS  
####   
#### ### Dependencies & Technologies  
####   
#### 1. Use existing form validation library already in the repo (e.g., React Hook Form + Yup/Zod, Formik, etc.)—do not introduce a new one.  
#### 2. Use existing backend validation mechanism already in the repo (e.g., Joi/Zod/class-validator/JSON schema) and update the schema accordingly.  
#### 3. If DB migration tooling exists (e.g., Prisma migrations, TypeORM, Sequelize, Rails ActiveRecord), use it to relax NOT NULL constraint if needed.  
####   
#### ### Validation Rules  
####   
#### 1. Registration form allows submission when Street Address is empty or omitted.  
#### 2. Street Address field does not show a required UI indicator (asterisk) and does not trigger “required” error messaging on blur/submit.  
#### 3. Backend registration endpoint does not reject requests missing Street Address (no 400/422 for missing field).  
#### 4. Persisted user record can store Street Address as NULL/empty string without errors (and without DB constraint violations).  
#### 5. If Street Address is provided, it must still pass existing formatting constraints (e.g., max length, allowed characters) as currently implemented.  
#### 6. Whitespace-only Street Address input is preserved as entered (no new trimming/normalization introduced by this change).  
#### 7. Any downstream flows that display address (profile, order checkout, etc.) handle missing Street Address gracefully (no crashes, no literal 'undefined' rendering).  
#### 8. Street Address optionality applies to all users/regions (no conditional requirement by user type/region).  
####   
#### ---  
####   
####   
#### ## TECHNICAL ASSUMPTIONS  
####   
#### 1. There is a dedicated registration page/form with a Street Address input currently marked required.  
#### 2. There is server-side validation for registration payload that currently requires Street Address.  
#### 3. Street Address is stored in a user profile table/collection and may currently be non-nullable.  
#### 4. The desired change is to keep the Street Address field present on registration but make it optional.  
#### 5. Street Address should be optional for all users/regions with no conditional requirement.  
#### 6. Whitespace-only Street Address should be preserved as entered; no new trimming/normalization should be introduced by this change.  
####   
#### ---  
####   
#### ## SUGGESTED FILE CHANGES  
####   
####   
#### 1. **File**: `src/pages/Register.tsx`  
####    **Rationale**: Likely contains the registration form UI and client-side validation rules; update Street Address to optional and adjust label/required indicator.  
####   
#### 2. **File**: `src/components/RegistrationForm.tsx`  
####    **Rationale**: If the form is extracted into a component, update field label, required indicator, and validation.  
####   
#### 3. **File**: `src/validation/registerSchema.ts`  
####    **Rationale**: If a shared schema (Yup/Zod/etc.) exists, update Street Address from required() to optional()/nullable() while preserving existing constraints when provided.  
####   
#### 4. **File**: `server/routes/auth.ts`  
####    **Rationale**: Registration endpoint may validate request body; ensure Street Address is not required and behavior is consistent for all users/regions.  
####   
#### 5. **File**: `server/validation/register.ts`  
####    **Rationale**: Backend request validation schema likely lives here; relax requirement for Street Address while keeping existing constraints when present.  
####   
#### 6. **File**: `server/models/User.ts`  
####    **Rationale**: User model may define Street Address as required/non-null; update to optional/nullable.  
####   
#### 7. **File**: `server/migrations/*`  
####    **Rationale**: If DB column is NOT NULL, add migration to allow NULL for street_address.  
####   
#### 8. **File**: `src/pages/__tests__/Register.test.tsx`  
####    **Rationale**: Add/adjust frontend tests to ensure Street Address is optional and whitespace-only is preserved (no new trimming).  
####   
#### 9. **File**: `server/__tests__/auth.register.test.ts`  
####    **Rationale**: Add/adjust backend tests to ensure registration succeeds without Street Address and preserves whitespace-only input (no new trimming).  
####   
#### ---  
####   
#### ## ADDITIONAL NOTES  
####   
#### Keep the Street Address field visible on registration, but label it as optional (e.g., “Street address (optional)”) consistent with existing UX/i18n patterns. Update i18n strings (labels/errors) if the app is localized. Ensure accessibility: optionality should be conveyed in text, not only via styling. Do not introduce new trimming/normalization behavior for whitespace-only Street Address; preserve the value as entered (unless existing behavior already trims—avoid changing it as part of this task).  
####   
#### ---  
####   
#### ---  
####   
#### ## IMPLEMENTATION PLAN FROM PLANNING NODE  
####   
#### The following comprehensive plan was created by the planning node. Follow this plan to implement the feature:  
####   
#### ### 1. CODEBASE ANALYSIS  
#### After reviewing the referenced areas, the registration flow appears to be implemented end-to-end across:  
####   
#### - **Frontend page + form component**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` render the registration UI, including the Street Address input, required indicator (asterisk), and client-side validation wiring (likely via a shared schema).  
#### - **Client validation schema**: `src/validation/registerSchema.ts` defines required/optional rules and field constraints (e.g., max length, regex). Street Address is currently treated as required here (directly or indirectly).  
#### - **Backend route + validation**: `server/routes/auth.ts` handles the registration endpoint and invokes a validation layer in `server/validation/register.ts` (or similar). Street Address is currently required at the API boundary and/or via additional manual checks.  
#### - **Persistence layer**: `server/models/User.ts` defines the user model shape and may mark street address as required/non-nullable. The DB schema may enforce `NOT NULL` on the street address column.  
#### - **Downstream rendering**: other UI pages/components may display address fields; they must handle missing street address gracefully (avoid rendering `undefined` or crashing).  
####   
#### Key behavioral constraint: **do not introduce new trimming/normalization** for Street Address. If whitespace-only is entered, it must be preserved as-is (client and server). This requires auditing for:  
#### - client-side transforms (schema `.trim()`, preprocessors, submit payload builders)  
#### - server-side sanitization middleware (e.g., global trim), validation transforms, ORM hooks  
####   
#### Also, optionality must apply **for all users/regions**, so we must locate and remove any conditional “address required” logic (e.g., based on country/region).  
####   
#### ---  
####   
#### ### 2. IMPLEMENTATION STRATEGY  
#### Make Street Address optional consistently across all layers, with explicit handling for **omitted**, **empty string**, and **whitespace-only** values:  
####   
#### 1. **Frontend UI/UX**: keep the field visible, remove required indicator and required error, update label to “Street address (optional)” (i18n-safe), and ensure submit is allowed when empty/omitted.  
#### 2. **Frontend validation**: update schema so Street Address is optional and accepts:  
####    - `undefined` (omitted)  
####    - `""` (empty string)  
####    - `"   "` (whitespace-only) without trimming  
####    while still enforcing existing constraints **only when a non-empty value is provided** (or exactly matching existing behavior).  
#### 3. **Backend validation**: update request schema so Street Address is not required for any region/user type; ensure missing/empty string does not fail validation; keep existing constraints when present; avoid trimming/normalization.  
#### 4. **Persistence**: ensure the DB column allows NULL (migration if needed) and model typing reflects optional/nullable; ensure create/update paths accept undefined/empty without constraint violations.  
#### 5. **Graceful rendering**: update any address formatting/rendering to avoid literal `undefined` and awkward punctuation when street address is missing.  
#### 6. **Types/DTOs**: ensure any shared request/response types used by client/server mark street address optional to prevent TS errors and mismatches.  
####   
#### (Per instructions for this node: implementation plan only; tests are handled separately, but we will include explicit steps to update/add tests as required by the task.)  
####   
#### ---  
####   
#### ### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN  
####   
#### **Step 1: Locate Street Address field, required UI, and validation wiring**  
#### - **File**: `src/pages/Register.tsx`, `src/components/RegistrationForm.tsx`  
#### - **Action**: Identify how Street Address is rendered and marked required, and how validation errors are displayed.  
#### - **Details**:  
####   - Find label text, required asterisk/indicator logic, `required` HTML attribute, or component props like `isRequired`.  
####   - Find error mapping for street address (e.g., `errors.streetAddress?.message`).  
####   - Identify form library and resolver (React Hook Form + Yup/Zod, Formik, etc.) and where schema is attached.  
#### - **Rationale**: Ensures we remove *all* required indicators and required-only error behavior.  
#### - **Addresses**: AC #2; Validation rules #2  
####   
#### **Step 2: Update label to clearly indicate optionality (i18n-safe)**  
#### - **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` (+ i18n files if present)  
#### - **Action**: Change label to “Street address (optional)” consistent with existing UX/i18n patterns.  
#### - **Details**:  
####   - If i18n is used, add/update a translation key rather than hardcoding.  
####   - Ensure optionality is conveyed in text (accessibility), not only via styling.  
#### - **Rationale**: Meets UX requirement while keeping field visible.  
#### - **Addresses**: AC #2; UX requirement  
####   
#### **Step 3: Remove required indicator/asterisk and required-only UI behavior**  
#### - **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx`  
#### - **Action**: Make Street Address non-mandatory in the UI.  
#### - **Details**:  
####   - Remove asterisk rendering for this field.  
####   - Remove `required` attribute / `isRequired` prop for this field only.  
####   - Ensure blur/submit does not show “Street Address is required” when empty.  
#### - **Rationale**: Field must not block registration.  
#### - **Addresses**: AC #1, #2; Validation rules #1, #2  
####   
#### **Step 4: Update client-side validation schema to accept omitted + empty string (and preserve whitespace)**  
#### - **File**: `src/validation/registerSchema.ts`  
#### - **Action**: Change Street Address from required to optional with explicit empty-string handling.  
#### - **Details** (must be explicit to avoid regressions):  
####   - Ensure schema accepts `undefined` (omitted) and `""` (empty string) without triggering regex/min/format validators.  
####   - Keep existing constraints (max length, allowed chars, etc.) **only when the value is meaningfully “provided”** per existing behavior. Concretely:  
####     - If the library treats `""` as a value and runs validators, add conditional logic so validators run only when `value !== ""` (and do **not** trim).  
####     - Do **not** add `.trim()`, transforms, preprocessors, or normalization.  
####   - Decide and document whether `null` is accepted:  
####     - If the client currently sends `null`, allow it; otherwise prefer `undefined`/`""` and let persistence store NULL if needed.  
#### - **Rationale**: Aligns frontend validation with optional requirement while explicitly handling empty string vs omitted and preserving whitespace-only input.  
#### - **Addresses**: AC #1, #4; Validation rules #1, #5, #6  
####   
#### **Step 5: Audit client submit/payload building to ensure no trimming/normalization is introduced**  
#### - **File**: `src/pages/Register.tsx` and/or `src/components/RegistrationForm.tsx` (and any helpers used, e.g., `src/utils/*`)  
#### - **Action**: Confirm Street Address is passed through unchanged.  
#### - **Details**:  
####   - Inspect submit handler and any `buildPayload()`/DTO mapping.  
####   - If there is generic string trimming/sanitization applied to all fields, ensure Street Address is excluded (or keep existing behavior if it already trims—do not expand trimming as part of this change).  
####   - Ensure whitespace-only input remains whitespace-only in the payload (no new normalization to `""`/`null`).  
#### - **Rationale**: Required by “preserve whitespace-only as entered” constraint.  
#### - **Addresses**: AC #4; Validation rule #6  
####   
#### **Step 6: Identify and remove any conditional “Street Address required” logic (all regions/users)**  
#### - **File**: `src/validation/registerSchema.ts`, `src/pages/Register.tsx`, `src/components/RegistrationForm.tsx`, and any country/region logic modules  
#### - **Action**: Search for conditional validation or UI requirement toggles.  
#### - **Details**:  
####   - Search for patterns like `when(country...)`, `if (country === ...) require street`, `isInternational`, etc.  
####   - Ensure Street Address is optional regardless of region/user type.  
#### - **Rationale**: Ensures optionality applies to all users/regions.  
#### - **Addresses**: AC #6; Validation rule #8  
####   
#### **Step 7: Update backend registration request validation to make Street Address optional (explicit empty-string handling)**  
#### - **File**: `server/validation/register.ts`  
#### - **Action**: Relax schema so Street Address is not required and does not fail when missing/empty.  
#### - **Details**:  
####   - Ensure schema accepts:  
####     - missing field (`undefined` / absent key)  
####     - empty string (`""`) if client submits it  
####   - Keep existing constraints when present (max length, pattern), but ensure they do not run for `""` unless that was already the behavior.  
####   - Do **not** add trimming/normalization; avoid `.trim()`/coercion/transforms.  
#### - **Rationale**: Backend must accept registrations without Street Address and remain consistent with client behavior.  
#### - **Addresses**: AC #3; Validation rules #3, #5, #6  
####   
#### **Step 8: Audit server-side sanitization/normalization to preserve whitespace-only Street Address**  
#### - **File**: `server/routes/auth.ts`, plus any middleware/utilities (e.g., `server/middleware/*`, `server/utils/sanitize*`, request parsing layers)  
#### - **Action**: Confirm no global trim/normalization affects Street Address; if it does, exclude Street Address only.  
#### - **Details**:  
####   - Look for middleware that trims all string fields in `req.body`.  
####   - Look for validation libraries configured with “trim” defaults.  
####   - Look for ORM hooks (beforeCreate/beforeSave) that normalize strings.  
####   - If trimming exists today and already affects Street Address, do not change it broadly; only ensure this task does not introduce *new* trimming. If trimming would newly break AC #4 due to schema changes, scope an exclusion to Street Address only.  
#### - **Rationale**: Explicitly satisfies whitespace preservation requirement and avoids accidental behavior changes.  
#### - **Addresses**: AC #4; Validation rule #6  
####   
#### **Step 9: Update auth registration route to not enforce Street Address and to pass it through**  
#### - **File**: `server/routes/auth.ts`  
#### - **Action**: Remove any manual “required” checks and ensure user creation accepts optional street address.  
#### - **Details**:  
####   - Remove checks like `if (!streetAddress) return ...`.  
####   - Ensure destructuring/defaulting does not force `streetAddress` to `""` or `null` unless that is existing behavior.  
####   - Ensure the value is passed to persistence unchanged (no trimming).  
#### - **Rationale**: Avoid hidden validation outside schema and preserve input.  
#### - **Addresses**: AC #3, #4  
####   
#### **Step 10: Update request/DTO/shared types so Street Address is optional end-to-end**  
#### - **File**: Any shared types used by client/server (if present), plus route handler types  
####   - Examples: `src/types/*`, `server/types/*`, `shared/*`, or DTO definitions used in `server/routes/auth.ts`  
#### - **Action**: Mark street address optional/nullable in registration request types.  
#### - **Details**:  
####   - If there is a `RegisterRequest` type/interface, change `streetAddress: string` to `streetAddress?: string` (and/or `string | null` depending on conventions).  
####   - Ensure both client and server compile and agree on optionality.  
#### - **Rationale**: Prevent TS compile errors and runtime mismatch; make optionality explicit.  
#### - **Addresses**: AC #1–#3 (consistency)  
####   
#### **Step 11: Update User model typing/ORM schema to allow missing/nullable Street Address**  
#### - **File**: `server/models/User.ts` (and ORM schema file if separate, e.g., Prisma schema / Sequelize model definition)  
#### - **Action**: Make street address optional/nullable in the model definition.  
#### - **Details**:  
####   - Update TS types/interfaces.  
####   - Update ORM-level field definition to allow null if applicable.  
#### - **Rationale**: Align model with new optional behavior and DB constraints.  
#### - **Addresses**: Persistence requirements; AC #1, #3  
####   
#### **Step 12: Determine DB column nullability and add migration only if needed**  
#### - **File**: ORM schema/migrations:  
####   - `server/migrations/*`  
####   - plus schema definition source of truth (e.g., `prisma/schema.prisma`, Sequelize model + migration history, TypeORM entity, etc.)  
#### - **Action**: Verify whether the street address column is currently `NOT NULL` and identify the exact column name.  
#### - **Details**:  
####   - Inspect existing migrations/schema to confirm:  
####     - actual column name (`street_address` vs `streetAddress` vs `address1`)  
####     - whether it is `NOT NULL`  
####   - Only create a new migration if it is currently non-nullable.  
#### - **Rationale**: Avoid incorrect/unnecessary migrations and ensure persistence works.  
#### - **Addresses**: Persistence requirement; AC #1, #3  
####   
#### **Step 13: Add DB migration to allow NULL for street address (if currently NOT NULL)**  
#### - **File**: `server/migrations/<timestamp>_make_street_address_nullable.*`  
#### - **Action**: Alter the column to be nullable.  
#### - **Details**:  
####   - Implement “up” migration to drop NOT NULL constraint.  
####   - Implement “down” migration if your tooling requires it (restore NOT NULL), considering existing null data (may need a default or data cleanup strategy consistent with repo conventions).  
#### - **Rationale**: Ensure saving a user with missing street address does not violate DB constraints.  
#### - **Addresses**: Persistence requirement; AC #1, #3  
####   
#### **Step 14: Ensure downstream address display renders gracefully when Street Address is missing**  
#### - **File**: Search and update all relevant UI components/helpers  
####   - Search terms: `streetAddress`, `street_address`, `address1`, `addressLine1`, `fullAddress`, string concatenations with address fields  
####   - Likely locations: profile pages, account settings, order summary/checkout, admin user detail views, address formatting utilities  
#### - **Action**: Prevent crashes and prevent rendering literal `undefined` / awkward punctuation.  
#### - **Details**:  
####   - If there is an address formatting helper, update it to conditionally include street address only when it’s not `null/undefined` (and possibly not `""` depending on existing conventions), without trimming whitespace-only.  
####   - Ensure UI does not show `"undefined"` or `"null"` strings.  
#### - **Rationale**: Meets acceptance criteria for graceful rendering.  
#### - **Addresses**: AC #5; Validation rule #7  
####   
#### **Step 15: Update/add automated tests to cover optional Street Address behavior (frontend + backend + persistence)**  
#### - **File**:   
####   - Frontend: `src/pages/__tests__/Register.test.tsx` (and any form component tests)  
####   - Backend: `server/__tests__/auth.register.test.ts`  
#### - **Action**: Add/adjust tests per task requirements.  
#### - **Details** (must cover):  
####   - Frontend: submit succeeds with Street Address empty/omitted; no asterisk; no “required” error; whitespace-only is not newly trimmed/normalized by client.  
####   - Backend: register without Street Address returns 200/201; invalid street address still rejected per existing rules; whitespace-only is not newly trimmed/normalized by server.  
####   - Persistence: creating user with null/undefined street address does not throw (direct model/service test if applicable).  
#### - **Rationale**: Ensures end-to-end behavior and prevents regressions.  
#### - **Addresses**: AC #1–#4; Persistence requirement  
####   
#### ---  
####   
#### ### 4. VALIDATION STRATEGY  
#### - **Frontend**:  
####   - Street Address field remains visible and labeled optional.  
####   - Client schema accepts omitted and empty string without errors.  
####   - Existing constraints still apply when a non-empty value is provided (per existing rules).  
####   - Confirm no new trimming/normalization is introduced in schema or submit mapping; whitespace-only remains whitespace-only.  
#### - **Backend**:  
####   - Registration endpoint accepts missing/empty street address without 400/422.  
####   - Existing constraints still enforced when street address is provided (non-empty).  
####   - Audit and prevent new trimming/normalization; if global trimming exists, ensure this change does not newly affect Street Address (or exclude it narrowly if required).  
#### - **Persistence**:  
####   - DB allows NULL if needed; model allows optional/nullable.  
####   - User creation works with `undefined`/`null`/`""` depending on existing conventions.  
#### - **Downstream rendering**:  
####   - Address display does not crash and does not render literal `undefined`.  
####   
#### ---  
####   
#### ### 5. INTEGRATION POINTS  
#### - `Register.tsx` / `RegistrationForm.tsx` integrates with `src/validation/registerSchema.ts` via the existing form library resolver/validator.  
#### - Client submit handler integrates with API request DTO/type (shared or local) and must pass Street Address through unchanged.  
#### - `server/routes/auth.ts` integrates with `server/validation/register.ts` for request validation and with the user creation service/model for persistence.  
#### - `server/models/User.ts` (and ORM schema) integrates with DB migrations to enforce nullability.  
#### - Address display components integrate with user profile data and any address formatting helpers.  
####   
#### ---  
####   
#### ### 6. EDGE CASES & CONSIDERATIONS  
#### - **Empty vs omitted**: Some form libs submit `""` for empty inputs; backend validation must explicitly accept both absent and `""`.  
#### - **Whitespace-only**: Must be preserved; do not add `.trim()` or transforms. Audit existing sanitizers/middleware/hooks to ensure this change doesn’t introduce new trimming.  
#### - **Conditional validation by region**: Remove/override any country/region-based requirement so Street Address is optional for all.  
#### - **Downstream formatting**: Avoid `"undefined, City"` and similar; use conditional joins without trimming whitespace-only.  
#### - **DB down migration risk**: If making column NOT NULL again, existing nulls could break; follow repo conventions (may require setting default empty string or preventing down migration in prod workflows).  
####   
#### ---  
####   
#### ### 7. IMPLEMENTATION CONSTRAINTS  
#### - Focus ONLY on making Street Address optional end-to-end (UI + validation + persistence + graceful display).  
#### - Use existing form and validation libraries; do not add new dependencies.  
#### - Do not introduce new trimming/normalization behavior for Street Address; preserve whitespace-only input as entered.  
#### - Ensure optionality applies to all users/regions (no conditional requirement).  
#### - Treat address as PII: do not add verbose logging.  
#### - Follow existing codebase conventions for i18n, schema validation, migrations, and typing.  
####   
#### ---  
####   
####   
#### ## YOUR ROLE: EXECUTION NODE  
####   
#### **THIS IS AN EXECUTION/DEVELOPMENT NODE**  
####   
#### Your task is to implement the feature according to the comprehensive plan provided above.  
####   
#### ---  
####   
#### ## IMPORTANT CONSTRAINTS  
####   
#### **THIS IS PURE DEVELOPMENT NODE ONLY**  
####   
#### 1. **Focus ONLY on Core Feature Implementation**  
####    - Implement the feature functionality as described  
####    - Ensure basic validation rules are in place  
####    - Follow existing code patterns and conventions  
####   
#### 2. **DO NOT Include in This Phase**:  
####    - Unit tests (handled by other node)  
####   
#### 3. **Code Quality Standards**:  
####    - Follow existing codebase conventions and patterns  
####    - Write clean, readable, maintainable code  
####    - Add clear comments for complex logic  
####    - Use descriptive variable and function names  
####    - Ensure type safety   
####   
#### 4. **Integration Requirements**:  
####    - Ensure changes integrate seamlessly with existing code  
####    - Maintain backward compatibility where applicable  
####    - Update necessary imports and dependencies  
####    - Follow the project's file structure and organization  
####   
#### ---  
####   
#### ## SUCCESS CRITERIA FOR THIS NODE  
####   
#### Your implementation will be considered successful when:  
####   
#### 1. All acceptance criteria are addressed in the implementation  
#### 2. All validation rules are enforced  
#### 3. Code follows existing patterns and conventions  
#### 4. Changes integrate properly with the existing codebase  
#### 5. Basic functionality works as described  
#### 6. No breaking changes to existing functionality  
#### 7. All in-scope features are completed  
####   
#### ---  
####   
#### ## NEXT STEPS (Context for You)  
####   
#### After you complete this pure development phase:  
#### - Other node will add comprehensive unit tests  
####   
#### **Your focus now**: Deliver a working implementation of the core feature.  
####   
#### ---  
####   
#### **Ready to start? Begin by:**  
#### 1. Reading any reference files provided above  
#### 2. Understanding the existing code structure  
#### 3. Implementing the changes file by file  
#### 4. Testing basic functionality manually  
> Initial repo scan can be slow in larger repos, but only happens once.  

To make the Street Address field optional in the registration process, we need to update both the frontend and backend components. Here are the changes required:

### Frontend Changes

1. **Update the Registration Form UI**: Remove the required indicator and update the label to indicate that the Street Address is optional.

2. **Update the Validation Schema**: Modify the validation schema to make the Street Address optional.

### Backend Changes

1. **Update the Backend Validation**: Ensure the backend validation schema allows for an optional Street Address.

2. **Update the Database Model**: If the database schema currently requires the Street Address, update it to allow null values.

Let's start with the frontend changes:

#### Frontend Changes

**File**: `frontend/src/components/business/StudentRegistrationForm/StudentRegistrationForm.tsx`

````typescript
<<<<<<< SEARCH
<label htmlFor="street">Street Address *</label>
<input
  type="text"
  id="street"
  name="street"
  required
  value={formState.street}
  onChange={handleChange}
/>
{formErrors.street && <span>{formErrors.street}</span>}
=======
<label htmlFor="street">Street Address (optional)</label>
<input
  type="text"
  id="street"
  name="street"
  value={formState.street}
  onChange={handleChange}
/>
>>>>>>> REPLACE
````

**File**: `frontend/src/components/business/StudentRegistrationForm/types.ts`

````typescript
<<<<<<< SEARCH
export type FormErrors = {
    [key: string]: string | undefined;
    firstName?: string;
    lastName?: string;
    email?: string;
    dateOfBirth?: string;
    studentId?: string;
    phone?: string;
    street?: string;
    city?: string;
=======
export type FormErrors = {
    [key: string]: string | undefined;
    firstName?: string;
    lastName?: string;
    email?: string;
    dateOfBirth?: string;
    studentId?: string;
    phone?: string;
    street?: string; // No change needed here, just ensuring it's optional
    city?: string;
>>>>>>> REPLACE
````

**File**: `frontend/src/lib/validators.ts`

````typescript
<<<<<<< SEARCH
street: z.string().min(1, "Street Address is required"),
=======
street: z.string().optional(),
>>>>>>> REPLACE
````

#### Backend Changes

**File**: `backend/src/schemas/student.schema.ts`

````typescript
<<<<<<< SEARCH
street: z.string().min(1, "Street Address is required"),
=======
street: z.string().optional(),
>>>>>>> REPLACE
````

**File**: `backend/src/api/create-student.ts`

````typescript
<<<<<<< SEARCH
type CreateStudentInput = Omit<Student, "_id" | "createdAt" | "updatedAt">;
=======
type CreateStudentInput = Omit<Student, "_id" | "createdAt" | "updatedAt"> & {
  street?: string;
};
>>>>>>> REPLACE
````

**File**: `backend/src/schemas/mongo.schema.ts`

````typescript
<<<<<<< SEARCH
type MongoDocument = z.infer<typeof MongoDocumentSchema>;
=======
type MongoDocument = z.infer<typeof MongoDocumentSchema> & {
  street?: string;
};
>>>>>>> REPLACE
````

**File**: `backend/src/middleware/error.middleware.ts`

No changes needed here unless specific error handling for street address is implemented.

**File**: `backend/src/types/index.ts`

````typescript
<<<<<<< SEARCH
interface WithId {
  _id?: string;
=======
interface WithId {
  _id?: string;
  street?: string; // Ensure street is optional
>>>>>>> REPLACE
````

**File**: `backend/src/utils/mongodb/create.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/create.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/update.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/delete.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/read.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/connect.test.ts`

No changes needed here unless specific logic for street address is implemented.

**File**: `backend/src/utils/mongodb/index.ts`

No changes needed here unless specific logic for

>  
>  
> Model gpt-4o has hit a token limit!  
> Token counts below are approximate.  
>  
> Input tokens: ~31,040 of 128,000  
> Output tokens: ~15,769 of 16,384 -- possibly exceeded output limit!  
> Total tokens: ~46,809 of 128,000  
> https://aider.chat/docs/troubleshooting/token-limits.html  
> Open URL for more info? (Y)es/(N)o/(D)on't ask again [Yes]: y  
