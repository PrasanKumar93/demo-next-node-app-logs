### 1. CODEBASE ANALYSIS
- **Backend validation**
  - Zod v3 schemas live in `backend/src/schemas/`.
  - Target file: `backend/src/schemas/student.schema.ts` (contains `StudentSchema`).
  - Current issue: `firstName` likely has `.min(1)` but **no `.trim()`** and **no `.max(6)`**.
  - **Integration risk**: It must be confirmed that the **create-student** route/controller actually validates incoming payloads with `StudentSchema` (directly or via a request validator middleware). If it doesn’t, updating the schema alone won’t satisfy acceptance criteria #2.

- **Frontend form + validation**
  - Student creation page: `frontend/src/app/students/new/page.tsx` (likely renders the form).
  - Form component: `frontend/src/components/business/StudentRegistrationForm/` (exact file name to confirm, e.g. `student-registration-form.tsx` or `StudentRegistrationForm.tsx`).
  - Frontend validation uses **Zod v4**, but the exact schema location is currently ambiguous:
    - Candidate: `frontend/src/lib/validators.ts`
    - Or a schema co-located under `frontend/src/components/business/StudentRegistrationForm/`
  - **Critical ambiguity to resolve**: identify the **exact schema export name** used by the form (e.g., `studentRegistrationSchema`) and how it’s wired (e.g., `react-hook-form` resolver vs manual `safeParse`).

- **TextInput wrapper / maxLength plumbing**
  - UI input wrapper: `frontend/src/components/ui/TextInput/`.
  - Need to confirm how to pass native input attributes:
    - direct `maxLength` prop?
    - `inputProps={{ maxLength: 6 }}` forwarded to underlying `<input>`?
    - MUI v5/v6 patterns like `slotProps={{ input: { ... }}}`?
  - **Decision criterion**: we must verify in the wrapper implementation that the rendered DOM `<input>` receives `maxLength="6"`; otherwise we must extend the wrapper.

- **Testing stack**
  - Backend: Vitest.
  - Frontend: Vitest + React Testing Library.
  - Task scope/acceptance criteria explicitly require tests for both FE and BE; the plan must include concrete test steps and file targets.

---

### 2. IMPLEMENTATION STRATEGY
Enforce the same rule in **three layers** to meet UX + correctness requirements:

1. **Backend (source of truth)**: Zod v3 `StudentSchema.firstName` must:
   - trim leading/trailing whitespace
   - require non-empty after trim
   - enforce max length **6 characters** after trim
   - return a clear error message for max length

2. **Frontend validation (immediate feedback + prevents bypass)**: Zod v4 schema used by the form must mirror the backend rule:
   - `.trim().min(1).max(6)`
   - ensure the form displays the error for `firstName` when invalid

3. **Frontend input constraint (UX guardrail)**:
   - set `maxLength={6}` on the actual DOM input so normal typing can’t exceed 6
   - still rely on schema validation to catch programmatic bypasses

Additionally:
- **Trimming must affect the submitted payload**: ensure the submit handler uses the **parsed/validated output** (trimmed) rather than raw form state.
- **Verify backend route uses StudentSchema**: if not, wire it in so API rejects invalid `firstName`.

---

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Update backend `StudentSchema.firstName` to trim + max length 6**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `StudentSchema` definition for `firstName`
- **Details**:
  - Change `firstName` to: `z.string().trim().min(1, <msg>).max(6, <msg>)`
  - Example messages (copy not strict):
    - min: `"First name is required"`
    - max: `"First name must be at most 6 characters"`
  - Ensure `.trim()` is placed **before** `.min()` and `.max()`.
- **Rationale**: Enforces server-side constraint and whitespace handling.
- **Addresses**: AC #2, #3, #4, #5; validation rules (trim then 1..6).

**Step 2: Verify/ensure create-student API request validation uses `StudentSchema`**
- **File**: (to locate) likely one of:
  - `backend/src/routes/students*.ts`
  - `backend/src/controllers/students*.ts`
  - `backend/src/middlewares/validate-request*.ts`
  - `backend/src/handlers/students*.ts`
- **Action**: Confirm the create endpoint validates request body with `StudentSchema` (or a schema that includes it).
- **Details**:
  - If already validated: no code change, but explicitly confirm the path (document in PR/notes).
  - If not validated: add/attach validation middleware or explicit `StudentSchema.parse(req.body)` (or `safeParse` with proper error response) in the create-student flow.
  - Ensure the error response includes a field-level error for `firstName` (whatever the repo’s standard error format is).
- **Rationale**: Without this, backend won’t reject invalid payloads, failing AC #2.
- **Addresses**: AC #2 (backend rejection), also supports #3/#4.

**Step 3: Add/adjust backend schema tests for `firstName` boundaries + trimming**
- **File**: `backend/src/schemas/student.schema.test.ts` (create if missing) or existing test file for this schema
- **Action**: Add Vitest tests (no mocks) targeting `StudentSchema`
- **Details (test cases)**:
  - Accepts trimmed length 1: `"A"`
  - Accepts trimmed length 6: `"ABCDEF"`
  - Rejects trimmed length 7: `"ABCDEFG"`
  - Trims leading/trailing whitespace: `" John "` parses to `"John"` (assert parsed output)
  - Rejects whitespace-only: `"   "` fails `.min(1)` after trim
- **Rationale**: Locks in backend behavior and prevents regressions.
- **Addresses**: AC #2, #3, #4, #5; test requirements.

**Step 4: Identify the exact frontend Zod schema used by StudentRegistrationForm and update it**
- **File**: Determine actual schema location by searching for:
  - `z.object({ firstName: ... })`
  - `StudentRegistration` schema export
  - `zodResolver(` usage
  - imports into `StudentRegistrationForm`
  - likely `frontend/src/lib/validators.ts` OR a file under `frontend/src/components/business/StudentRegistrationForm/`
- **Action**: Update the *actual* schema used by the form (not a duplicate)
- **Details**:
  - Update `firstName` rule to: `.string().trim().min(1, <msg>).max(6, <msg>)`
  - Ensure the schema is the one wired into the form validation (resolver or submit parse).
  - Keep error copy clear; ideally align with backend message but not required.
- **Rationale**: Ensures immediate client-side feedback and blocks bypassed maxLength.
- **Addresses**: AC #1, #3, #4, #5.

**Step 5: Ensure frontend submit flow uses parsed (trimmed) values**
- **File**: `frontend/src/components/business/StudentRegistrationForm/<actual-form-file>.tsx`
- **Action**: Ensure the API payload uses trimmed output from Zod validation
- **Details** (depending on implementation):
  - If using `react-hook-form` + `zodResolver`:
    - Confirm the resolver is configured to use the Zod schema that includes `.trim()`.
    - Ensure the submit handler uses the `data` provided by `handleSubmit` (which should be the validated shape).
    - If the current setup does not apply Zod transforms, add an explicit `schema.parse(data)` in submit before calling API and use the parsed result.
  - If using manual validation:
    - Replace raw values with `schema.parse(formValues)` output before submit.
- **Rationale**: Guarantees `" John "` becomes `"John"` before sending to backend; consistent FE/BE.
- **Addresses**: AC #3, #5.

**Step 6: Add `maxLength=6` to the firstName input in StudentRegistrationForm**
- **File**: `frontend/src/components/business/StudentRegistrationForm/<actual-form-file>.tsx`
- **Action**: Apply a DOM-level max length constraint to the firstName field
- **Details**:
  - If the form uses the shared `TextInput` wrapper:
    - Use the wrapper’s supported API:
      - `maxLength={6}` if it exists, OR
      - `inputProps={{ maxLength: 6 }}` if forwarded, OR
      - `slotProps` pattern if the wrapper is MUI-based and uses MUI v6.
  - Confirm it applies only to `firstName`.
- **Rationale**: Meets “should not accept more than 6 characters” UX requirement.
- **Addresses**: AC #1.

**Step 7: If needed, extend `TextInput` wrapper to forward maxLength to the DOM input**
- **File**: `frontend/src/components/ui/TextInput/<actual-textinput-file>.tsx` (confirm exact filename)
- **Action**: Add support for passing `maxLength` (or `inputProps`) through to the underlying input element
- **Details**:
  - Verification criterion: after change, RTL test can query the textbox and assert it has `maxLength="6"`.
  - Implement in the repo’s established pattern (prop typing, forwarding, consolidated exports).
- **Rationale**: Ensures the maxLength attribute actually lands on the DOM input.
- **Addresses**: AC #1.

**Step 8: Add/adjust frontend tests for maxLength + schema enforcement + trimming**
- **File**: `frontend/src/components/business/StudentRegistrationForm/student-registration-form.test.tsx` (or existing test file; follow repo naming)
- **Action**: Add RTL + Vitest tests (no mocks) covering UI and validation behavior
- **Details (test cases)**:
  1. **Input has maxLength=6**
     - Render the form, locate firstName input (label/testid), assert `toHaveAttribute('maxLength', '6')`.
  2. **Typing >6 is constrained**
     - Use `userEvent.type(input, 'ABCDEFG')`, assert `input.value.length === 6`.
  3. **Bypass maxLength and submit triggers validation error and blocks submit**
     - Programmatically set value to 7 chars (e.g., `fireEvent.change` with `'ABCDEFG'` if typing is constrained).
     - Submit form with other required fields valid.
     - Assert error message shown for firstName and that submit handler/API call did not run (use real form behavior; if the form calls a module function, assert it wasn’t called without mocking by checking UI state change or disabled loading state doesn’t occur—use repo’s existing testing pattern).
  4. **Trimming passes**
     - Enter `" John "` plus other required valid fields, submit, assert no firstName error.
     - If the UI shows submitted/next state, assert it proceeds; or assert the payload uses trimmed value if the form exposes it (depending on existing patterns).
- **Rationale**: Proves UX guardrail + schema correctness + trimming.
- **Addresses**: AC #1, #3, #4, #5; test requirements.

**Step 9: Ensure error messaging is clear and displayed**
- **File**: `frontend/src/components/business/StudentRegistrationForm/<actual-form-file>.tsx` (and possibly form field error component)
- **Action**: Confirm `firstName` max-length error is rendered near the field
- **Details**:
  - Ensure the form maps Zod/react-hook-form errors to the TextInput error display.
  - If errors are currently not shown for this field, wire it consistently with other fields.
- **Rationale**: Acceptance criteria requires user-visible feedback when invalid.
- **Addresses**: AC #1, #4.

**Step 10: Documentation + deployment checklist items**
- **File**: (repo-specific; e.g., `CHANGELOG.md`, `docs/`, or PR template)
- **Action**: Add brief task documentation and note deployment considerations
- **Details**:
  - Document: “firstName trimmed length must be 1..6” and that both FE/BE enforce it.
  - Deployment: note that API will now reject longer first names; no DB migration.
- **Rationale**: Task checklist requires documentation and deployment notes.
- **Addresses**: Checklist items.

---

### 4. VALIDATION STRATEGY
- **Canonical rule** (both FE and BE): `firstName` is a string whose **trimmed** length is **1..6**.
- **Frontend**
  - Prevent typical invalid input via `maxLength=6`.
  - Enforce correctness via Zod v4 schema (`.trim().min(1).max(6)`).
  - Display field-level error message for >6 and whitespace-only.
- **Backend**
  - Enforce correctness via Zod v3 schema (`.trim().min(1).max(6)`).
  - Ensure create-student endpoint actually uses the schema and returns a field error for `firstName`.

---

### 5. INTEGRATION POINTS
- **Backend**
  - `StudentSchema` in `backend/src/schemas/student.schema.ts` must be used by the create-student request validation path (middleware/controller).
  - Error formatting should follow existing API conventions; ensure `firstName` is identifiable in the error response.

- **Frontend**
  - `frontend/src/app/students/new/page.tsx` renders the form; no change unless schema is defined there.
  - `StudentRegistrationForm` must:
    - use the updated Zod schema (resolver or manual parse)
    - pass `maxLength` to the actual DOM input (possibly via `TextInput`)
    - submit trimmed values (parsed output)

---

### 6. EDGE CASES & CONSIDERATIONS
- **“letters” vs “characters”**: implement **max 6 characters** (as per acceptance criteria and technical requirements). Use “characters” wording in error messages to avoid ambiguity.
- **Whitespace handling**:
  - `" John "` should validate as `"John"` (trim before length checks).
  - `"   "` should be invalid (becomes empty after trim).
- **No silent truncation on submit**:
  - UI maxLength prevents typing beyond 6, but if bypassed, schema must reject and show error (FE) and API must reject (BE).
- **Internal spaces**: `"Jo hn"` counts toward length; only leading/trailing whitespace is trimmed.
- **Consistency**: Keep FE/BE messages reasonably aligned; not required but reduces confusion.

---

### 7. IMPLEMENTATION CONSTRAINTS
- Focus ONLY on implementing the max-length + trim validation across FE/BE, wiring `maxLength` in UI, and adding the required tests.
- Follow existing codebase conventions:
  - Arrow functions only
  - Consolidated exports at end of file
  - Hyphenated filenames
  - `export type` separately
- Ensure type safety and consistent error handling.
- No DB migrations; existing records with longer names are out of scope.
- Tests must be added/adjusted as specified (backend Vitest + frontend RTL/Vitest), without mocks, following repo patterns.