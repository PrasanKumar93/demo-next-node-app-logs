### 1. CODEBASE ANALYSIS
Aider should first locate where `streetAddress` is defined/validated across the stack and identify all “required” enforcement points. Expect three layers:

- **Frontend UI/forms**
  - Registration page and a reusable registration form component (likely `src/pages/Registration.*`, `src/components/forms/StudentRegistrationForm.*`).
  - Student edit/update page and form component (likely `src/pages/**/EditStudent.*`, `src/components/forms/StudentEditForm.*`).
  - A shared validation schema (likely `src/validation/*student*.*`) using a library such as Yup/Zod or custom validators.
  - “Required” may be enforced via:
    - schema rules (`required()`, `min(1)`, `nonempty()`)
    - field props (`required`, `aria-required`, asterisk rendering)
    - submit button disable logic (e.g., `isValid` depends on required fields)

- **Backend API validation**
  - Create and update endpoints (likely `server/routes/*student*.*` and/or controllers).
  - Request validators/DTO schemas (likely `server/validators/*student*.*`).
  - “Required” may be enforced via:
    - schema validators (Joi/Zod/class-validator)
    - explicit checks in controller/service
    - model-level validations

- **Persistence / DB schema**
  - Student model definition (likely `server/models/*Student*.*` or ORM schema).
  - DB migration history (`migrations/*`) to see if `street_address` / `streetAddress` is `NOT NULL`.
  - Ensure the column allows `NULL` (and that empty string is also acceptable per existing conventions).

Key behavioral constraint to preserve:
- **No trimming/normalization changes**: do not add `.trim()`, `.transform()`, or middleware that converts whitespace-only to empty/null. If such trimming already exists, do not expand it as part of this task; only remove “required” constraints.

---

### 2. IMPLEMENTATION STRATEGY
Make `streetAddress` optional consistently in **all** create/update flows by:
1. Removing “required” indicators and client-side required validation for `streetAddress` in registration and edit forms.
2. Updating frontend validation schemas so `streetAddress` is optional but still respects existing constraints **when provided** (e.g., max length).
3. Updating backend request validation/DTOs so `streetAddress` is optional on both create and update; accept missing, `null`, and `""`.
4. Ensuring persistence allows `NULL` (migration only if needed) and model-level validations do not require presence.
5. Keep API behavior: if provided (including whitespace-only), store and return it unchanged.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Inventory all `streetAddress` touchpoints**
- **Files**: (discovered via repo search)
  - `src/**` for `streetAddress`, `street address`, `street_address`
  - `server/**` for the same
  - `migrations/**` and ORM schema files
- **Action**: Identify every place where `streetAddress` is marked required or validated as non-empty.
- **Details**:
  - Find schema rules like `required`, `nonempty`, `min(1)`, `presence: true`, `NOT NULL`.
  - Find UI props like `required`, `aria-required`, label asterisks.
- **Rationale**: Ensures optionality is consistent across registration + edit/update and avoids mismatched validation.
- **Addresses**: All acceptance criteria indirectly (completeness).

**Step 2: Update registration UI field to not be required**
- **File**: `src/pages/Registration.*` (or wherever the registration form is composed)
- **Action**: Remove any required marker/prop for Street Address.
- **Details**:
  - If the input component receives `required`, set it to false/remove it.
  - Remove `aria-required="true"` if present.
  - Remove asterisk rendering for this field only (keep for other required fields).
  - Ensure submit button enablement does not depend on streetAddress being non-empty.
- **Rationale**: UI must not block submission or show required indicator.
- **Addresses**: AC1, AC3; Validation rules UI #3.

**Step 3: Update registration form component validation**
- **File**: `src/components/forms/StudentRegistrationForm.*`
- **Action**: Make `streetAddress` optional in the form’s validation rules.
- **Details** (follow existing validation library pattern):
  - If Yup: change from `yup.string().required(...)` to `yup.string().notRequired()` (and keep `.max(...)` etc.).
  - If Zod: change from `z.string().min(1)`/`nonempty()` to `z.string().optional()` or `z.union([z.string(), z.undefined()])`; if `null` is allowed, use `.nullable()` as well.
  - Ensure empty string `""` is accepted (some schemas treat `""` as invalid if `min(1)` exists).
  - Do **not** add `.trim()` or transforms.
- **Rationale**: Client-side validation must allow empty/missing street address.
- **Addresses**: AC1, AC3; Validation rules #1, #4, #9.

**Step 4: Update edit/update UI field to not be required**
- **File**: `src/pages/**/EditStudent.*`
- **Action**: Remove required indicator/props for Street Address in edit page.
- **Details**:
  - Same as registration: remove `required`, `aria-required`, asterisk.
  - Ensure clearing the field does not block save.
- **Rationale**: Edit flow must allow clearing street address.
- **Addresses**: AC2, AC3; Validation rules UI #3.

**Step 5: Update edit/update form component validation**
- **File**: `src/components/forms/StudentEditForm.*`
- **Action**: Make `streetAddress` optional in edit/update validation schema.
- **Details**:
  - If update schema differs from create schema, update both.
  - Ensure both “omit field” and “set to empty string” pass validation.
  - Preserve any max length/format checks when a non-empty value is provided.
  - No trimming/transforms.
- **Rationale**: Save must succeed when field is cleared or omitted.
- **Addresses**: AC2, AC3; Validation rules #2, #4, #9.

**Step 6: Update shared frontend student validation schema(s)**
- **File**: `src/validation/*student*.*` (or equivalent shared schema module)
- **Action**: Make `streetAddress` optional in any shared create/update schemas.
- **Details**:
  - If there is a single schema used by both forms, update it once.
  - If there are separate `createStudentSchema` and `updateStudentSchema`, update both.
  - Ensure schema accepts:
    - missing key
    - `""`
    - `null` (only if frontend sends null; otherwise optional is enough)
  - Keep existing constraints (e.g., max length) without adding normalization.
- **Rationale**: Prevent regressions where one form uses shared schema and still enforces required.
- **Addresses**: AC1–AC3; Validation rules #1–#4, #9.

**Step 7: Update backend create-student request validation**
- **File**: `server/validators/*student*.*` (create DTO/schema)
- **Action**: Mark `streetAddress` optional and allow empty string.
- **Details**:
  - Remove `required`/presence constraints.
  - Ensure validator does not reject `""`.
  - If validator supports `null`, allow it if API may receive it.
  - Do not add trimming/transform steps.
- **Rationale**: Backend must accept missing/empty streetAddress on create.
- **Addresses**: AC1, AC3; Validation rules #5, #7, #9.

**Step 8: Update backend update-student request validation**
- **File**: `server/validators/*student*.*` (update DTO/schema)
- **Action**: Mark `streetAddress` optional and allow empty string.
- **Details**:
  - For PATCH-like semantics: omission should mean “no change”; explicit `""` should mean “set empty string”.
  - Ensure schema allows both.
  - No trimming/transform.
- **Rationale**: Backend must accept clearing streetAddress and not error when omitted.
- **Addresses**: AC2, AC3; Validation rules #6, #7, #9.

**Step 9: Ensure route/controller/service does not enforce required streetAddress**
- **Files**: `server/routes/*student*.*` and any controller/service modules they call
- **Action**: Remove any manual checks that reject missing/empty streetAddress.
- **Details**:
  - Look for code like `if (!streetAddress) return 400`.
  - Ensure mapping logic includes `streetAddress` when present, and safely handles undefined/null.
  - For update: ensure the update payload can set `streetAddress` to `""` (don’t filter falsy values with patterns like `if (streetAddress) obj.streetAddress = streetAddress`).
- **Rationale**: Even with schema changes, controller logic can still block or drop empty strings.
- **Addresses**: AC1–AC4; Validation rules #5–#7, #9.

**Step 10: Update model-level validation (if any)**
- **File**: `server/models/*Student*.*` (or ORM model definition)
- **Action**: Remove presence/required validation for street address.
- **Details**:
  - If using ORM validations (e.g., Sequelize `allowNull: false`, Rails `validates :street_address, presence: true`), change to allow null and remove presence validation.
  - Keep other validations intact.
- **Rationale**: Prevent persistence-layer rejection.
- **Addresses**: AC1–AC4; Validation rules #8.

**Step 11: Update DB schema to allow NULL (migration only if needed)**
- **Files**: `migrations/*` (new migration if current schema is NOT NULL)
- **Action**: Alter street address column to be nullable.
- **Details**:
  - Detect actual column name (`street_address` vs `streetAddress`) and table.
  - Create migration using existing migration tool conventions.
  - Only change nullability; do not modify defaults or add triggers that trim.
- **Rationale**: Inserts/updates must not fail due to NOT NULL constraint.
- **Addresses**: AC1–AC2; Validation rules #8.

**Step 12: Verify API response/serialization preserves whitespace**
- **Files**: Any serialization/mapping layer (often in controller/service)
- **Action**: Ensure no `.trim()` or normalization is applied to `streetAddress`.
- **Details**:
  - If there is a global sanitizer, do not change it; just ensure this task doesn’t introduce new trimming.
  - Ensure `streetAddress: "   "` round-trips.
- **Rationale**: Explicit requirement to preserve whitespace-only values.
- **Addresses**: AC4; Validation rules #9.

**Step 13: Update any inline docs/UI helper text that claims it’s required**
- **Files**: Any form label/help text modules, possibly `src/components/forms/*` or localization files
- **Action**: Remove “required” wording for Street Address only.
- **Details**:
  - Keep other required field messaging unchanged.
- **Rationale**: Consistency and avoids user confusion.
- **Addresses**: Supports AC3.

---

### 4. VALIDATION STRATEGY
Implementation should satisfy validations by ensuring:

- **Frontend**
  - Street Address field has no required UI attributes/indicators.
  - Form schema allows `streetAddress` to be `undefined` or `""` without errors.
  - Any “build payload” logic includes empty string when user clears it (no falsy filtering).

- **Backend**
  - Create/update validators treat `streetAddress` as optional and accept `""` (and `null` if applicable).
  - Controllers/services do not reject or drop empty string values.
  - Persistence accepts NULL (and empty string) without constraint violations.

- **No trimming**
  - No new `.trim()`/transform introduced in either frontend or backend for this field.

---

### 5. INTEGRATION POINTS
- **Frontend form library integration**: Update the schema/resolver and field props in the same way other optional fields are handled in the repo.
- **API contract**: Ensure request DTOs and controllers align so omission vs explicit empty string behaves correctly (especially for update).
- **ORM/DB**: Model nullability and DB column nullability must match to avoid runtime errors.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Update semantics**:
  - Omitted `streetAddress` should not overwrite existing value.
  - Explicit `streetAddress: ""` should clear it (set to empty string) if that’s what UI sends.
- **Null vs empty string**:
  - UI may send `""` when cleared; some APIs may send `null`. Backend should accept both if feasible without breaking conventions.
- **Max length/format**:
  - Keep existing constraints when a non-empty value is provided; don’t accidentally loosen/tighten.
- **Fનલsy filtering bug**:
  - Common issue: `if (streetAddress) ...` drops `""`. Must use checks like `if (streetAddress !== undefined)`.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on making Street Address optional in registration + edit/update flows.
- DO NOT add unit tests in this node.
- Follow existing codebase conventions for validation, forms, DTOs, and migrations.
- Maintain backward compatibility and do not change auth/authz.
- Do not introduce trimming/normalization; preserve whitespace-only values exactly.