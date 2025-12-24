### 1. CODEBASE ANALYSIS
Goal: find every place where `streetAddress` (and variants) is enforced as required across **UI**, **frontend validation**, **API validation**, **controller/service mapping**, **ORM/model**, and **DB schema**, and ensure whitespace-only values are not trimmed/coerced.

**Repo-wide discovery (must do before edits)**
- Search terms (case-insensitive) across `src/`, `server/`, `migrations/`, and any `shared/` or `packages/` folders:
  - `streetAddress`, `street_address`, `street address`
  - common alternates: `addressLine1`, `address1`, `line1`, `street`, `addr1`
  - validation keywords near those fields: `required`, `nonempty`, `min(1)`, `presence`, `allowNull`, `NOT NULL`, `trim`, `transform`, `sanitize`
- Identify:
  1. Registration form component(s) and edit/update form component(s)
  2. Shared frontend validation schema(s) used by both flows
  3. Backend request validators/DTOs for create + update
  4. Any controller/service “manual required checks”
  5. Any payload-building logic that drops falsy values (especially `""`)
  6. ORM schema/model constraints and DB column nullability
  7. Any sanitization middleware (frontend or backend) that trims/coerces strings
  8. API contract types/docs (OpenAPI, TS types, generated clients) that mark `streetAddress` required

**DB nullability verification (explicit)**
- Determine the actual storage field and table:
  - Inspect ORM schema files (e.g., `prisma/schema.prisma`, `sequelize` model definitions, `typeorm` entities, etc.)
  - Inspect existing migrations for the student table creation/alter statements
  - If available in repo tooling, check schema snapshot or SQL definition
- Confirm whether the column is `NOT NULL` and whether there are constraints/triggers that modify values.

---

### 2. IMPLEMENTATION STRATEGY
Make `streetAddress` optional **consistently** for both **create** and **update** flows by changing three layers in lockstep:

1. **Frontend UI + validation**
   - Remove required indicator/props (`required`, `aria-required`, asterisk) for Street Address in registration and edit forms.
   - Update frontend validation schema(s) so `streetAddress` accepts: `undefined` (omitted), `null`, and `""` (empty string).
   - Preserve existing constraints **only when a value is provided** (e.g., max length), without adding trimming/normalization.
   - Ensure payload builders do not drop `""` (fix truthy checks).

2. **Backend API validation + mapping**
   - Update create and update request validators/DTOs to accept `streetAddress` as optional and allow `null` and `""`.
   - Remove any manual “required” checks in controllers/services.
   - Ensure update mapping supports:
     - omission => do not change existing value
     - explicit `""` or `null` => clear/set accordingly
   - Ensure no new trimming/normalization is introduced; audit existing sanitizers.

3. **Persistence/DB**
   - Align ORM/model nullability with DB nullability.
   - If DB column is `NOT NULL`, add a migration to allow `NULL` (only if needed).
   - Ensure serialization/deserialization does not coerce whitespace-only to empty/null.

Also: update API contract/docs/types if they still mark `streetAddress` required, and include documentation + deployment steps per checklist.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Repo-wide inventory of Street Address field and constraints**
- **File**: N/A (discovery step)
- **Action**: Perform targeted searches and map all touchpoints.
- **Details**:
  - Search in `src/**` for field rendering, required props, label asterisks, and schema usage.
  - Search in `server/**` for validators/DTOs, controllers, services, and any `if (!streetAddress)` checks.
  - Search for payload builders that do `if (value) obj.key = value` (drops `""`).
  - Search for sanitizers: `.trim()`, `.transform()`, `sanitize`, `normalize`, middleware that mutates request bodies.
  - Search for API contract definitions: `openapi.*`, `swagger.*`, `api-types.*`, `StudentCreateRequest`, etc.
- **Rationale**: Prevent missing a required enforcement point or a coupled validation.
- **Addresses**: Completeness for AC1–AC5; Validation rules #1–#10.

**Step 2: Identify and verify DB column/table and nullability**
- **File**: ORM schema + migrations (e.g., `prisma/schema.prisma`, `server/models/Student.*`, `migrations/*`)
- **Action**: Determine whether the DB enforces `NOT NULL` for street address and the exact column name.
- **Details**:
  - Locate the student table definition and the street address column (`street_address` vs `streetAddress`).
  - Confirm ORM field config (`nullable`/`allowNull`) and DB constraint (`NOT NULL`).
  - Record whether DB accepts `NULL` today; if not, plan migration in Step 11.
- **Rationale**: Ensures persistence won’t reject inserts/updates when streetAddress is missing/null.
- **Addresses**: AC1–AC2; Validation rule #8.

**Step 3: Update registration UI to remove required indicator/props**
- **File**: `src/pages/Registration.*` and/or `src/components/forms/StudentRegistrationForm.*` (actual paths from Step 1)
- **Action**: Make Street Address visually and semantically optional.
- **Details**:
  - Remove `required` prop and `aria-required="true"` for Street Address input.
  - Remove any asterisk/required label rendering for this field.
  - Ensure any “required fields” helper text doesn’t list Street Address.
- **Rationale**: UI must not block submission or mislead users.
- **Addresses**: AC1, AC3; Validation rule UI #3.

**Step 4: Update shared/reusable Field/Input component behavior (if applicable)**
- **File**: `src/components/*Field*.*`, `src/components/*Input*.*`, design system components (as found)
- **Action**: Ensure Street Address isn’t implicitly marked required by schema introspection or default props.
- **Details**:
  - If the component auto-adds `required` based on schema metadata, ensure Street Address metadata is optional.
  - If required-asterisk is driven by a `requiredFields` list, remove Street Address from that list.
- **Rationale**: Prevent hidden required indicators even after form-level changes.
- **Addresses**: AC3; Validation rule UI #3.

**Step 5: Update frontend validation schema for create (registration)**
- **File**: `src/validation/*student*.*` and/or schema co-located with registration form
- **Action**: Make `streetAddress` optional and accept `undefined`, `null`, and `""`.
- **Details** (adapt to actual library):
  - Remove `required()` / `nonempty()` / `min(1)` constraints.
  - Keep existing constraints that should apply when provided (e.g., `.max(…)`), ensuring they do not reject `""` unless that was explicitly desired (it is not for this task).
  - Explicitly allow `null` if the schema currently rejects it.
  - Do **not** add `.trim()` or transforms.
- **Rationale**: Client-side validation must allow empty/missing street address.
- **Addresses**: AC1, AC3; Validation rules #1, #4, #9.

**Step 6: Update edit/update UI to remove required indicator/props**
- **File**: `src/pages/**/EditStudent.*` and/or `src/components/forms/StudentEditForm.*`
- **Action**: Make Street Address optional in edit/update screens.
- **Details**:
  - Remove `required` and `aria-required` for Street Address.
  - Remove required asterisk/label indicator for this field.
  - Ensure clearing the field does not disable Save due to “required” logic.
- **Rationale**: Users must be able to clear Street Address and save.
- **Addresses**: AC2, AC3; Validation rule UI #3.

**Step 7: Update frontend validation schema for update (edit/save)**
- **File**: `src/validation/*student*.*` (update schema) and/or edit form schema
- **Action**: Make `streetAddress` optional for update and accept `undefined`, `null`, and `""`.
- **Details**:
  - For PATCH semantics: omission should be valid; explicit empty string should be valid.
  - Preserve existing constraints when non-empty values are provided (e.g., max length).
  - No trimming/transforms.
- **Rationale**: Update must succeed when field is omitted or cleared.
- **Addresses**: AC2, AC3; Validation rules #2, #4, #9.

**Step 8: Fix frontend payload construction to not drop empty string**
- **File**: API client layer / submit handlers (e.g., `src/api/*`, `src/services/*`, form submit functions)
- **Action**: Ensure `streetAddress: ""` is sent when user clears it; omission only when truly untouched/undefined.
- **Details**:
  - Replace patterns like `if (streetAddress) payload.streetAddress = streetAddress` with `if (streetAddress !== undefined) ...`
  - Ensure `null` is passed through if UI uses null to clear.
  - Confirm registration payload can omit the field entirely when blank (also acceptable).
- **Rationale**: Prevent “falsy filtering” from breaking AC2/AC4.
- **Addresses**: AC2, AC4; Validation rules #6–#7, #9.

**Step 9: Update backend create-student request validation to accept missing/null/empty**
- **File**: `server/validators/*student*.*` (create schema/DTO)
- **Action**: Mark `streetAddress` optional and allow `null` and `""`.
- **Details**:
  - Remove required/presence constraints.
  - Ensure validator accepts:
    - missing key
    - `streetAddress: ""`
    - `streetAddress: null`
  - Keep existing max length/format constraints for non-empty values only (avoid rejecting `""`).
  - Do not add trimming/transform.
- **Rationale**: Backend must not reject create requests without street address.
- **Addresses**: AC1, AC3; Validation rules #5, #7, #9.

**Step 10: Update backend update-student request validation to accept missing/null/empty**
- **File**: `server/validators/*student*.*` (update schema/DTO)
- **Action**: Mark `streetAddress` optional and allow `null` and `""`.
- **Details**:
  - Ensure omission is valid (no change).
  - Ensure explicit `""` and `null` are valid (clear/set).
  - Preserve other required fields behavior (do not loosen unrelated requirements).
  - No trimming/transform.
- **Rationale**: Backend must accept clearing street address and not error when omitted.
- **Addresses**: AC2, AC3; Validation rules #6–#7, #9, #10.

**Step 11: Update backend controller/service mapping to avoid manual required checks and falsy filtering**
- **File**: `server/routes/*student*.*`, controllers/services used by create/update
- **Action**: Remove any manual “streetAddress required” logic and ensure mapping preserves empty/whitespace.
- **Details**:
  - Remove checks like `if (!streetAddress) return 400`.
  - Fix update mapping patterns like:
    - `if (body.streetAddress) update.streetAddress = body.streetAddress` (drops `""`)
    - Replace with `if ('streetAddress' in body) update.streetAddress = body.streetAddress`
  - Ensure create mapping passes through `""`, `null`, and whitespace-only unchanged.
- **Rationale**: Even with validator changes, controller logic can still reject or drop values.
- **Addresses**: AC1–AC4; Validation rules #5–#7, #9.

**Step 12: Align ORM/model validation with optional streetAddress**
- **File**: `server/models/*Student*.*` or ORM entity/schema files
- **Action**: Ensure model does not enforce presence and allows null.
- **Details**:
  - Remove model-level presence validation (e.g., Rails `presence: true`, Sequelize `allowNull: false`).
  - Ensure ORM field is nullable if DB is nullable.
  - Ensure serialization/deserialization does not coerce `null`/`""`/whitespace.
- **Rationale**: Prevent persistence-layer rejection and unintended coercion.
- **Addresses**: AC1–AC4; Validation rules #7–#9.

**Step 13: Add DB migration if (and only if) street address is NOT NULL**
- **File**: `migrations/*` (new migration)
- **Action**: Alter the student street address column to allow NULL.
- **Details**:
  - Use the project’s migration tool conventions.
  - Only change nullability; do not add defaults, triggers, or transforms.
  - Ensure down migration restores prior state if required by repo standards.
- **Rationale**: DB must accept inserts/updates without street address.
- **Addresses**: AC1–AC2; Validation rule #8.

**Step 14: Audit existing sanitization/normalization to ensure whitespace-only is preserved**
- **File**: Any request sanitizers/middleware, frontend input normalizers, API client interceptors
- **Action**: Confirm no trimming/coercion is applied to `streetAddress`.
- **Details**:
  - If there is an existing global trim middleware, do not change it broadly in this task; instead ensure `streetAddress` is not newly added to trimming lists.
  - Verify mapping layers do not call `.trim()` on address fields.
- **Rationale**: Requirement explicitly demands whitespace-only values round-trip unchanged.
- **Addresses**: AC4; Validation rule #9.

**Step 15: Update API contract types/docs/comments that mark streetAddress required**
- **File**: OpenAPI/Swagger specs, shared TS types, API docs markdown, inline comments
- **Action**: Make `streetAddress` optional in contracts/documentation.
- **Details**:
  - Update request schema: remove from `required` list; allow `null` if supported.
  - Ensure generated client types (if any) reflect optionality.
- **Rationale**: Prevent consumer mismatch and future regressions.
- **Addresses**: Supports AC1–AC3; aligns with task notes.

**Step 16: Tests planning handoff (do not implement here)**
- **File**: N/A (planning step only; implementation in test node)
- **Action**: Enumerate concrete test cases to add/update for FE + BE.
- **Details**:
  - Frontend: registration submit with empty streetAddress; edit save after clearing; no required indicator/aria-required.
  - Backend: create without streetAddress; create with `""`; create with `null`; update omitting; update with `""`; update with `null`.
  - Whitespace-only preservation: `"   "` stored/returned unchanged.
  - Regression: other required fields still fail.
- **Rationale**: Task scope includes tests; this plan must explicitly cover them even if implemented elsewhere.
- **Addresses**: AC1–AC5; Validation rules #1–#10.

**Step 17: Task documentation + deployment steps**
- **File**: `CHANGELOG.md` / release notes / internal docs (whatever repo uses), deployment runbook
- **Action**: Document behavior change and deployment requirements.
- **Details**:
  - Document that Street Address is now optional for create/update.
  - If migration added: include migration execution steps and rollback notes.
  - Note no trimming/normalization changes.
- **Rationale**: Satisfy checklist items (documentation + deployment) and reduce rollout risk.
- **Addresses**: Checklist compliance; operational readiness.

---

### 4. VALIDATION STRATEGY
After implementation (and later tests), verify:

**Frontend**
- Street Address field:
  - no required asterisk
  - no `required` attribute
  - no `aria-required=true`
- Registration:
  - submit succeeds with streetAddress omitted and with `""`
- Edit/update:
  - save succeeds when streetAddress cleared to `""` (and/or set to `null` if UI uses null)
- Payload inspection:
  - clearing field results in request containing `streetAddress: ""` (not dropped)
- Whitespace:
  - entering `"   "` remains `"   "` in payload (no trim)

**Backend**
- Create endpoint accepts:
  - missing `streetAddress`
  - `streetAddress: ""`
  - `streetAddress: null`
- Update endpoint accepts:
  - omission (no change)
  - explicit `""` and `null` (clears/sets)
- Ensure no 400/422 due solely to streetAddress missing/empty.
- Confirm response/DB readback returns the same value (including whitespace-only).

**DB/ORM**
- If streetAddress is null, insert/update succeeds (no NOT NULL violation).
- ORM schema matches DB nullability.

---

### 5. INTEGRATION POINTS
- **Form library ↔ schema resolver**: ensure optionality is reflected both in UI props and schema rules.
- **Frontend submit handler ↔ API client**: ensure payload builder does not drop `""` and supports omission vs explicit clear.
- **API validator ↔ controller/service**: validator must allow optional; controller must not re-enforce required or filter falsy.
- **Service ↔ ORM update semantics**: update logic must distinguish:
  - field absent (do nothing)
  - field present with `""`/`null` (set/clear)
- **ORM ↔ DB migrations**: nullability aligned; migration applied in deployment.
- **API contract/docs ↔ implementation**: required lists/types updated to match behavior.

---

### 6. EDGE CASES & CONSIDERATIONS
- **PATCH vs PUT semantics**: If update endpoint is PUT-like (full replace), ensure missing streetAddress doesn’t inadvertently null it unless that’s current behavior; align with existing conventions.
- **Falsy filtering**: Must fix both FE and BE mapping to not drop `""`.
- **Null vs empty string**: Accept both on backend; store as provided (do not coerce).
- **Whitespace-only**: Must be accepted and preserved; audit for any existing trim middleware that could violate this.
- **Coupled address validation**: Ensure city/state/zip are not conditionally required based on streetAddress (unless already intended); do not change other fields’ requiredness.
- **Max length/format**: Keep existing constraints for non-empty values; ensure they don’t accidentally run on `null`/`undefined`.
- **PII logging**: Do not add logs that print streetAddress.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on making Street Address optional in registration + edit/update flows.
- DO NOT write actual unit/integration tests in this node (handled separately), but the plan must specify them.
- Follow existing codebase conventions for validation, forms, DTOs, migrations, and docs.
- Maintain backward compatibility and do not change auth/authz.
- Do not introduce trimming/normalization; preserve whitespace-only values exactly.
- Ensure type safety (update TS types/contracts where applicable) and avoid widening unrelated validations.