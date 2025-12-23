### 1. CODEBASE ANALYSIS
Aider should first locate and understand the current registration flow end-to-end (UI → client validation → API payload → backend request validation → persistence). Specifically, identify:

- **Registration UI component(s)**: likely `frontend/**/Registration*.tsx` (could be multiple variants: `RegistrationPage`, `RegistrationForm`, `Register`).
- **Form library + validation**: determine whether the form uses **React Hook Form + Zod/Yup**, **Formik + Yup**, or a custom validator. Find the schema/rules that currently mark `streetAddress` (or similar key) as required, including any **country/region conditional requirement**.
- **Shared validation schemas**: check `frontend/**/validation/**` for reusable address/user schemas; confirm how trimming/normalization is done (e.g., `.trim()`, `transform`, `setValueAs`, `preprocess`).
- **i18n strings**: search `frontend/**/i18n/**` for “Street Address is required” or similar.
- **API client + payload construction**: find where the registration request body is built; confirm whether it always sends `streetAddress` (possibly as `""`) or omits it when empty.
- **Backend registration endpoint**: locate `backend/**/register*.*` and any DTO/schema validators in `backend/**/validators/**`. Identify whether `streetAddress` is required by:
  - schema requiredness (e.g., Joi/Zod required)
  - decorators (e.g., `@IsNotEmpty`)
  - model-level constraints
- **Backend normalization**: confirm existing conventions for optional strings (store `null`, store `""`, or omit). Also confirm whether there is a global trim pipe/middleware.

Deliverable from this analysis step: a list of the exact field name(s) used (`streetAddress`, `address1`, `street`, etc.), where required markers are rendered, and where required validation is enforced.

---

### 2. IMPLEMENTATION STRATEGY
Make Street Address optional consistently across the stack:

1. **UI**: remove required indicator for Street Address (no asterisk, no “required” label, and do not add “(optional)”).
2. **Client validation**: update schema/rules so Street Address is **never required**, regardless of country/region/user type. Keep existing constraints when provided (max length/format).
3. **Normalization**: treat whitespace-only input as empty by trimming; ensure trimmed empty passes validation.
4. **API payload**: ensure registration can be submitted when Street Address is blank; payload may omit the field or send `""`/`null` per existing conventions—either must be accepted.
5. **Backend validation**: accept missing or empty Street Address; keep constraints when provided (length/format). Normalize whitespace-only to empty/null consistent with backend patterns.
6. **No regressions**: do not change requiredness of other fields.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Locate all Street Address field usages and requiredness**
- **File(s)**: (discovered) `frontend/**/Registration*.tsx`, `frontend/**/validation/**`, `backend/**/register*.*`, `backend/**/validators/**`
- **Action**: Search for the Street Address field key and required markers/validators.
- **Details**:
  - Find the input component rendering Street Address and how “required” is indicated (prop like `required`, `isRequired`, label rendering with `*`, etc.).
  - Find client schema rules: `.required()`, `.min(1)`, `nonempty()`, `@IsNotEmpty`, etc.
  - Find any conditional logic tied to country/region.
- **Rationale**: Ensures all enforcement points are updated (UI + validation + backend).
- **Addresses**: All acceptance criteria indirectly by ensuring complete coverage.

**Step 2: Update registration UI to remove required indicator**
- **File**: `frontend/**/Registration*.tsx` (exact file(s) found in Step 1)
- **Action**: Remove required marker/flag for Street Address only.
- **Details**:
  - If label is built from a shared component (e.g., `<FormLabel required>`), set it to non-required for Street Address.
  - If the asterisk is appended manually, remove it for this field.
  - Do **not** add “(optional)”.
  - Ensure accessibility attributes remain correct (e.g., `aria-required` should not be set for Street Address).
- **Rationale**: Meets UX requirement and avoids misleading required cues.
- **Addresses**: Acceptance criteria #2; Requirements (UI).

**Step 3: Make Street Address optional in client-side validation schema**
- **File**: `frontend/**/validation/**` (schema used by registration)
- **Action**: Change Street Address from required to optional, removing any conditional requirement.
- **Details** (apply based on library):
  - **Yup**: replace `.required()` with `.notRequired()` / `.optional()`; keep `.max(...)`, `.matches(...)` etc. Ensure `.trim()` is applied and empty string is allowed.
  - **Zod**: use `z.string().trim().max(...).optional()` OR `z.preprocess(trimToUndefined, z.string().max(...).optional())`. If schema currently uses `z.string().min(1)`, remove the `min(1)` requirement.
  - Remove any `when(country === ...)` / conditional `.required()` logic.
  - Ensure whitespace-only becomes empty (see Step 4).
- **Rationale**: Prevents client-side blocking and ensures optional across all regions.
- **Addresses**: Acceptance criteria #1, #3, #4; Requirements (client validation).

**Step 4: Ensure whitespace-only Street Address is treated as empty (client)**
- **File**: `frontend/**/Registration*.tsx` and/or `frontend/**/validation/**` (where normalization is done)
- **Action**: Normalize input by trimming; convert whitespace-only to empty/undefined consistently.
- **Details**:
  - Prefer existing pattern in codebase:
    - React Hook Form: `register('streetAddress', { setValueAs: v => v?.trim() || '' })` or `undefined` depending on conventions.
    - Schema transform: `.transform(v => v?.trim() ?? v)` and then allow `''`.
  - Ensure that after trimming, `""` does not trigger required validation (since it’s optional).
- **Rationale**: Meets explicit whitespace-only requirement and avoids accidental “non-empty” strings.
- **Addresses**: Acceptance criteria #1, #3; Validation rules (whitespace-only treated as empty).

**Step 5: Update API payload construction to not force Street Address**
- **File**: `frontend/**/Registration*.tsx` or `frontend/**/api/**` (where request body is built)
- **Action**: Ensure submission works when Street Address is blank; do not inject required placeholder.
- **Details**:
  - If payload builder currently always includes `streetAddress: values.streetAddress` and backend expects it, keep it but ensure it can be `""`/`null`.
  - If codebase convention is to **omit empty optional fields**, then:
    - after trimming, if empty, delete/omit `streetAddress` from payload.
  - Keep behavior unchanged for other fields.
- **Rationale**: Avoids backend validation errors and aligns with “absent or empty string” acceptance.
- **Addresses**: Acceptance criteria #1, #5.

**Step 6: Update any frontend i18n/copy implying Street Address is required**
- **File**: `frontend/**/i18n/**`
- **Action**: Remove/adjust strings like “Street Address is required”.
- **Details**:
  - If the string is only used for validation errors, it should no longer be referenced once requiredness is removed.
  - Keep other address validation messages (e.g., max length) intact.
- **Rationale**: Prevents stale messaging and ensures no “required” error appears.
- **Addresses**: Acceptance criteria #2.

**Step 7: Make Street Address optional in backend request validation**
- **File**: `backend/**/register*.*` and/or `backend/**/validators/**`
- **Action**: Relax backend validation to accept missing/empty Street Address.
- **Details** (apply based on backend framework):
  - If using DTO decorators:
    - remove `@IsNotEmpty()` / `@IsDefined()` for street address
    - add `@IsOptional()` and keep `@MaxLength(...)` / pattern validators
    - ensure validators don’t run on `""` if convention treats empty as “missing”; if needed, transform `""` → `null`/`undefined` before validation.
  - If using Joi/Zod:
    - mark field optional and allow `""` (e.g., `.optional().allow('')`) while still applying max length when non-empty (often via custom rule or `.empty('')` + `.default(undefined)`).
  - Remove any country-based requirement logic.
- **Rationale**: Backend must not reject requests without Street Address.
- **Addresses**: Acceptance criteria #5.

**Step 8: Backend normalization for whitespace-only / empty string**
- **File**: `backend/**/register*.*` (controller/service) or shared normalization middleware
- **Action**: Trim street address; convert whitespace-only to `null`/`""` per existing conventions.
- **Details**:
  - If there is a shared “trim strings” pipe/middleware, ensure it applies to `streetAddress`.
  - If not, add a localized normalization in registration handler: `streetAddress = streetAddress?.trim(); if (!streetAddress) streetAddress = null` (or `""`).
  - Ensure no new logging of address content is introduced.
- **Rationale**: Meets whitespace-only requirement and keeps stored data consistent.
- **Addresses**: Acceptance criteria #6; Technical requirements (normalization).

**Step 9: Ensure persistence layer tolerates missing Street Address**
- **File**: backend user/profile creation service/model file(s) (discovered)
- **Action**: Confirm create/update logic doesn’t assume non-null street address.
- **Details**:
  - If DB schema has NOT NULL constraint, adjust application-level mapping to store `""` if that’s the convention (since migrations are out of scope unless required to prevent runtime errors).
  - Ensure downstream code doesn’t crash when address is null/empty.
- **Rationale**: Prevent runtime errors after validation passes.
- **Addresses**: Acceptance criteria #6; “no regressions”.

**Step 10: Update any shared “address required” validators used by registration**
- **File**: `frontend/**/validation/**` and `backend/**/validators/**`
- **Action**: If there is a shared “AddressSchema” used in multiple places, ensure registration uses an optional variant or parameter.
- **Details**:
  - Prefer creating/using a schema option like `AddressSchema({ streetRequired: false })` if pattern exists.
  - Avoid changing other flows (checkout/shipping) that may still require street address.
- **Rationale**: Keeps registration optional without breaking other flows.
- **Addresses**: Acceptance criteria #3; Out-of-scope protection.

---

### 4. VALIDATION STRATEGY
- **Client**:
  - Street Address: optional; trim; if provided (non-empty after trim), enforce existing constraints (max length/format).
  - Ensure no conditional requiredness by country/region.
- **Backend**:
  - Accept `streetAddress` as **undefined**, **null** (if allowed), or **""**.
  - Trim and normalize whitespace-only to empty/null.
  - If non-empty, enforce existing constraints.
- **Regression safety**:
  - Do not modify requiredness of other fields; keep their validators intact.

---

### 5. INTEGRATION POINTS
- **Form UI ↔ validation schema**: ensure the UI required marker matches schema requiredness (Street Address not required in both).
- **Form values ↔ API payload**: ensure payload builder doesn’t reintroduce requiredness (e.g., forcing a non-empty placeholder).
- **API payload ↔ backend DTO/schema**: backend must accept both omitted and empty string.
- **Backend validation ↔ persistence**: ensure normalized value is compatible with DB/model expectations.

---

### 6. EDGE CASES & CONSIDERATIONS
- **Whitespace-only input**: must be trimmed and treated as empty on both FE and BE.
- **Conditional address rules**: remove any “street required for US/CA/etc.” logic specifically for registration.
- **Shared address schema used elsewhere**: avoid weakening requirements for checkout/shipping if they rely on the same schema—use a registration-specific variant/config.
- **PII handling**: do not add logs or error reporting that includes street address content.
- **Empty string vs null**: follow existing conventions; backend should accept both absent and `""` regardless.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on core feature implementation (no tests in this node).
- Do NOT introduce new form/validation libraries.
- Follow existing codebase conventions for:
  - schema composition
  - trimming/normalization
  - API client payload shaping
  - backend DTO/schema validation patterns
- Maintain backward compatibility: Street Address still saved/validated when provided; other required fields unchanged.