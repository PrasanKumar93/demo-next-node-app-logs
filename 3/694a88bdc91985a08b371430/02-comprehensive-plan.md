### 1. CODEBASE ANALYSIS
Goal: identify every place where “Street Address” is (a) visually marked required, (b) programmatically required (client + server), (c) normalized/trimmed, and (d) mapped into the API payload / persistence.

1. **Locate the registration UI entry points**
   - Search for registration pages/forms:
     - `Registration*.tsx`, `Register*.tsx`, `SignUp*.tsx`, `CreateAccount*.tsx`
   - Identify the exact field key(s) used for Street Address:
     - common variants: `streetAddress`, `street_address`, `address1`, `addressLine1`, `street`, `line1`
   - Identify how “required” is expressed in the UI:
     - label components: `<Label required />`, `<FormLabel isRequired />`, `required` prop on input wrapper
     - manual asterisk in label text
     - accessibility flags: `required` attribute, `aria-required`, `aria-invalid` patterns

2. **Determine the client form + validation stack and where rules live**
   - Confirm whether the form uses:
     - React Hook Form + Zod/Yup
     - Formik + Yup
     - custom validators
   - Find the schema(s) used by registration:
     - `frontend/**/validation/**`, `schemas/**`, `validators/**`
   - Specifically locate:
     - any `.required()`, `.min(1)`, `nonempty()`, `refine(v => v.length > 0)` on street address
     - any **conditional requiredness** (e.g., `when(country === 'US')`, `superRefine` based on region)

3. **Identify existing normalization/trimming conventions (commit to repo pattern)**
   - Find whether trimming happens via:
     - schema transforms (`.trim()`, `transform`, `preprocess`)
     - form registration (`setValueAs`, `valueAs`)
     - a shared utility (`normalizeString`, `trimToNull`, etc.)
     - a global request/response middleware
   - Record the established convention for optional strings:
     - omit field vs send `""` vs send `null`

4. **Inspect API client payload construction**
   - Locate the registration API call:
     - `frontend/**/api/**`, `services/**`, `mutations/**`
   - Determine whether payload currently:
     - always includes street address
     - forces a non-empty placeholder
     - converts empty to `null`/`undefined`
   - Decide which convention to follow (must match existing patterns elsewhere in the codebase).

5. **Locate backend registration endpoint + request validation**
   - Find controller/route/handler:
     - `backend/**/register*.*`, `auth/**`, `users/**`
   - Identify validation framework:
     - DTO decorators (class-validator), Joi, Zod, Yup, Rails validations, etc.
   - Find where street address is required:
     - schema “required” list
     - decorators like `@IsNotEmpty`, `@IsDefined`
     - custom validators that enforce address completeness

6. **Verify persistence constraints to avoid runtime errors**
   - Check DB schema/migrations/models for street address column constraints:
     - `NOT NULL` constraints
     - default values
     - length constraints
   - If `NOT NULL` exists, confirm existing convention is storing `""` (or adjust mapping to comply) without introducing migrations unless absolutely required to prevent runtime errors.

7. **Check centralized error-message mapping + i18n**
   - Search for:
     - “Street Address is required”
     - generic required-field mapping keyed by field name
   - Ensure no stale required message can surface via a generic “required” error handler.

8. **Quick telemetry/analytics schema check (no new logging)**
   - Search for events referencing address requiredness or validation codes:
     - e.g., `streetAddressRequired`, `address_missing`, `validation_error_field`
   - Ensure removing requiredness doesn’t break event schemas (and do not add PII logging).

Deliverable from analysis: exact field key, all UI required markers/attributes, client schema location, payload shaping location, backend validator location, and DB constraint status.

---

### 2. IMPLEMENTATION STRATEGY
Make Street Address optional **consistently** across UI, client validation, API payload, backend validation, and persistence—without weakening other flows (checkout/shipping) that may still require an address.

Key strategy points:
1. **UI**: remove *both* visual required marker and programmatic required attributes for Street Address only. Do not add “(optional)”.
2. **Client validation**: Street Address is **never required**, with **no country/region conditional requirement**. Keep existing constraints when provided.
3. **Normalization**: follow the repo’s existing trimming/normalization mechanism; ensure whitespace-only becomes empty and is accepted.
4. **Payload convention**: choose the existing convention used for optional fields (omit vs `""` vs `null`) and apply it consistently for street address.
5. **Backend validation**: accept street address being **absent** or **empty string**; ensure validators don’t fail on `""`/whitespace-only (common pitfall).
6. **Persistence**: ensure user creation does not crash when street address is missing/empty; comply with DB constraints using existing conventions.
7. **No regressions**: do not change requiredness of other fields; avoid weakening shared address schemas used outside registration by introducing a registration-specific variant if needed.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Inventory all Street Address touchpoints (UI → client schema → payload → backend → DB)**
- **File**: repo-wide search
- **Action**: Identify exact field name(s) and all enforcement points.
- **Details**:
  - Search for keys: `streetAddress`, `address1`, `addressLine1`, `street_address`, etc.
  - Find UI required markers and attributes (`required`, `aria-required`).
  - Find client schema rules and any conditional requiredness.
  - Find API payload builder and whether it omits/normalizes empty fields.
  - Find backend DTO/schema and any custom “address required” validators.
  - Check DB schema/migrations/models for nullability/defaults.
- **Rationale**: Prevents partial fixes where one layer still blocks submission.
- **Addresses**: Acceptance #1–#6 (enables complete coverage)

**Step 2: Update registration UI to remove required indicator + required attributes**
- **File**: `frontend/**/Registration*.tsx` (and any shared field components used there)
- **Action**: Make Street Address visually and programmatically optional.
- **Details**:
  - Remove asterisk/required marker for Street Address label.
  - Ensure the input does **not** set:
    - `required` HTML attribute
    - `aria-required="true"`
  - Keep other fields unchanged.
  - Do **not** add “(optional)” text.
- **Rationale**: Meets UX requirement and avoids accessibility mismatch (screen readers).
- **Addresses**: Acceptance #2; UI requirements

**Step 3: Make Street Address optional in the client validation schema (no conditional requirement)**
- **File**: `frontend/**/validation/**` (registration schema)
- **Action**: Remove requiredness for Street Address in all scenarios.
- **Details**:
  - Remove `.required()`, `.min(1)`, `nonempty()`, or equivalent.
  - Remove any `when(country/region)` logic that makes it required.
  - Keep existing constraints when provided (e.g., `.max(n)`, allowed chars).
  - If schema is shared with other flows that still require street address:
    - create a **registration-specific schema/variant** (e.g., `RegistrationAddressSchema` or `AddressSchema({ streetRequired: false })`)
    - ensure checkout/shipping continues using the required variant.
- **Rationale**: Ensures Street Address is optional for all regions without regressions elsewhere.
- **Addresses**: Acceptance #1, #3, #4; Validation rules #1, #3, #7

**Step 4: Implement whitespace-only normalization on the client using the repo’s existing pattern**
- **File**: whichever location Step 1 reveals is the canonical normalization point:
  - `frontend/**/validation/**` (preferred if schema-based)
  - or `frontend/**/Registration*.tsx` (if form-level `setValueAs` is the established pattern)
- **Action**: Trim Street Address and treat whitespace-only as empty.
- **Details**:
  - First, confirm existing convention (from analysis) and apply the same:
    - **Schema-based**: `trim` + transform/preprocess so `"   "` becomes `""` or `undefined` (whichever the repo uses).
    - **Form-based**: `setValueAs: v => normalize(v)` where normalize trims and returns empty when whitespace-only.
  - Ensure that after normalization, the optional schema accepts the empty value.
- **Rationale**: Explicit requirement; prevents “invisible” non-empty strings.
- **Addresses**: Acceptance #1, #3; Validation rule #8

**Step 5: Align frontend payload construction with existing optional-field convention**
- **File**: `frontend/**/api/**` or registration submit handler in `frontend/**/Registration*.tsx`
- **Action**: Ensure payload does not reintroduce requiredness and is consistent.
- **Details**:
  - Decide based on repo convention:
    - If the codebase typically **omits** empty optional fields: do not include `streetAddress` when normalized empty.
    - If it typically sends empty strings: include `streetAddress: ""` when empty.
    - If it uses `null`: send `null`.
  - Do not force a placeholder value.
  - Keep other fields unchanged.
- **Rationale**: Prevents inconsistent API behavior and avoids backend validation failures.
- **Addresses**: Acceptance #1, #5; Technical requirements (payload consistency)

**Step 6: Remove/adjust any frontend i18n/copy that implies Street Address is required**
- **File**: `frontend/**/i18n/**` and any centralized validation message maps
- **Action**: Ensure no “required” message can appear for Street Address.
- **Details**:
  - Remove/stop referencing “Street Address is required” strings.
  - Check centralized “required field” mapping keyed by field name; ensure Street Address is not included.
  - Keep other validation messages (max length/format) intact.
- **Rationale**: Avoids stale UX and ensures acceptance criterion #2.
- **Addresses**: Acceptance #2; Validation rule #2

**Step 7: Update backend request validation to make Street Address optional and accept empty string**
- **File**: `backend/**/register*.*`, `backend/**/validators/**` (actual paths discovered in Step 1)
- **Action**: Relax backend validation for Street Address only.
- **Details** (framework-agnostic requirements):
  - Mark field optional (accept **absent**).
  - Explicitly accept `""` (empty string) as valid input.
  - Remove any `notEmpty`/`defined` requirement.
  - Keep existing constraints when provided (max length/format).
  - **Pitfall handling**: many validators treat “optional” as “undefined only” and still validate `""`.
    - Add normalization (Step 8) *before* validation or configure schema to treat `""` as empty/missing.
- **Rationale**: Backend must not reject missing/empty street address.
- **Addresses**: Acceptance #5; Validation rules #5, #7

**Step 8: Backend normalization: trim and convert whitespace-only to empty/null per existing conventions**
- **File**: backend request pipeline (DTO transform / middleware / controller)
- **Action**: Ensure `"   "` does not trigger validators and is stored consistently.
- **Details**:
  - Apply trimming to street address.
  - If trimmed value is empty:
    - either set to `null`, `""`, or `undefined` based on existing backend conventions and DB constraints.
  - Ensure this happens in the same place other string fields are normalized (do not invent a new pattern if one exists).
  - Do not add logging of street address.
- **Rationale**: Meets whitespace-only requirement and prevents accidental validation failures.
- **Addresses**: Acceptance #5, #6; Validation rules #5, #8; Security/PII note

**Step 9: Ensure persistence layer tolerates missing/empty Street Address (verify DB constraints explicitly)**
- **File**: backend user/profile creation service + DB schema/migrations/models
- **Action**: Prevent runtime errors when street address is omitted.
- **Details**:
  - If DB column is nullable: store `null` (or omit) per convention.
  - If DB column is `NOT NULL`: store `""` (or default) per convention to avoid migration (unless absolutely required to prevent runtime errors).
  - Confirm downstream code that reads address does not assume non-null.
- **Rationale**: Ensures successful registration results in stable stored data and no crashes.
- **Addresses**: Acceptance #6; Technical requirement #6

**Step 10: Check analytics/telemetry compatibility (no new events, no PII)**
- **File**: `frontend/**/analytics/**`, `backend/**/telemetry/**` (if present)
- **Action**: Ensure event schemas don’t break due to requiredness change.
- **Details**:
  - If events include a list of missing required fields, ensure Street Address is not included.
  - If there is a boolean like `streetAddressRequired`, update it accordingly or remove usage.
  - Do not add any logging of street address values.
- **Rationale**: Prevents runtime/monitoring regressions and respects PII constraints.
- **Addresses**: “Additional notes” requirement; No-regressions

---

### 4. VALIDATION STRATEGY
**Client-side**
- Street Address:
  - optional in all cases (no locale conditions)
  - trim/normalize whitespace-only to empty
  - if non-empty after trim, enforce existing constraints (max length/format)
- Other fields: unchanged requiredness and validations.

**Server-side**
- Accept `streetAddress` as:
  - absent (`undefined` / missing key)
  - empty string `""`
  - whitespace-only (normalized to empty/null before validation/persistence)
- If non-empty after trim, enforce existing constraints (max length/format).
- Ensure validation framework does not mistakenly validate `""` as a provided value unless intended.

**Persistence**
- Store `null`/`""`/omitted according to existing conventions and DB constraints.
- Ensure no downstream assumptions break.

---

### 5. INTEGRATION POINTS
1. **UI required marker ↔ client schema**
   - UI must not show required marker or set `required/aria-required` if schema is optional.
2. **Client normalization ↔ schema**
   - Trimming must occur before validation (or within schema) so whitespace-only is treated as empty.
3. **Client payload ↔ backend validator**
   - Payload convention (omit vs `""` vs `null`) must be accepted by backend.
4. **Backend normalization ↔ backend validation**
   - Normalize `""`/whitespace-only so optional validators don’t fail on empty values.
5. **Backend validation ↔ persistence**
   - Stored value must satisfy DB constraints and not crash downstream reads.
6. **Telemetry**
   - Any “missing required fields” logic must not include Street Address.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Whitespace-only input**: must be trimmed and treated as empty on both FE and BE.
- **Empty string vs undefined vs null**: backend must accept absent and `""`; storage should follow existing conventions and DB constraints.
- **Shared address schema**: do not weaken checkout/shipping; use a registration-specific schema/variant if shared.
- **Validator pitfalls**: “optional” often doesn’t skip validation for `""`; ensure `""` is treated as empty/missing or explicitly allowed.
- **Accessibility**: remove `required` and `aria-required` for Street Address only; keep other fields correct.
- **No PII logging**: do not add logs containing street address; keep existing sanitization.
- **Country/region toggles**: ensure no conditional logic reintroduces requiredness when user changes country.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on core feature implementation.
- DO NOT include unit/integration tests in this node (handled separately).
- Do NOT introduce new form/validation libraries.
- Follow existing codebase conventions for:
  - trimming/normalization
  - schema composition/variants
  - API payload shaping
  - backend validation/DTO patterns
- Maintain backward compatibility:
  - Street Address still saved and validated when provided
  - other required fields remain required
- Ensure type safety and code quality; avoid broad refactors outside registration/address requiredness.