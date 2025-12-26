### 1. CODEBASE ANALYSIS

- **Backend validation** is handled with **Zod v3** schemas in `backend/src/schemas/`. The relevant file is:
  - `backend/src/schemas/student.schema.ts` → currently `firstName: z.string().min(1, "First name is required")` with **no trimming** and **no max length**.
- **Frontend** is a Next.js App Router app. Student creation page is at:
  - `frontend/src/app/students/new/page.tsx` (likely renders the form)
  - `frontend/src/components/business/StudentRegistrationForm/` (form implementation)
- **Frontend validation** uses **Zod v4** and is likely centralized in:
  - `frontend/src/lib/validators.ts` (per folder structure doc), or possibly a schema co-located in the form folder.
- **UI inputs** are wrapped in reusable components:
  - `frontend/src/components/ui/TextInput/` (likely wraps MUI TextField and supports `inputProps` / `maxLength`)
- Repo conventions to follow:
  - Arrow functions only
  - Consolidated exports at end of file
  - Hyphenated filenames
  - `export type` separately
  - Tests are co-located, but **tests are out of scope for this planning node** (another node will implement them)

### 2. IMPLEMENTATION STRATEGY

Enforce the same rule in three layers to satisfy acceptance criteria and prevent bypasses:

1. **Backend Zod schema (source of truth for API)**: trim then validate `firstName` length `1..6`.
2. **Frontend form schema (immediate feedback)**: same trim + `1..6` rule using Zod v4.
3. **Frontend input constraint (UX guardrail)**: set `maxLength={6}` (or `inputProps={{ maxLength: 6 }}` depending on wrapper) so users can’t type beyond 6 characters.

Trimming must happen **before** min/max checks, so whitespace-only becomes empty and fails `.min(1)`.

### 3. COMPREHENSIVE STEP-BY-STEP IMPLEMENTATION PLAN

**Step 1: Update backend StudentSchema firstName validation**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Modify `StudentSchema.firstName`
- **Details**:
  - Change from: `z.string().min(1, "First name is required")`
  - To: `z.string().trim().min(1, "First name is required").max(6, "First name must be at most 6 characters")`
  - Keep message copy simple/clear (exact copy not strict).
  - Ensure `.trim()` is placed before `.min()`/`.max()`.
- **Rationale**: Enforces the constraint server-side even if UI is bypassed; ensures whitespace is ignored for length.
- **Addresses**: Acceptance criteria #2, #3, #4, #5; validation rules (required after trim, <=6, whitespace-only invalid).

**Step 2: Ensure backend schema export pattern remains consistent**
- **File**: `backend/src/schemas/student.schema.ts`
- **Action**: Confirm exports remain consolidated at end
- **Details**:
  - Keep `export { StudentSchema };` and `export type { Student };` at the end (already matches convention).
  - No inline exports introduced.
- **Rationale**: Maintain repo coding preferences.
- **Addresses**: Implementation constraints / code quality.

**Step 3: Locate and update the frontend Zod schema used by StudentRegistrationForm**
- **File**: `frontend/src/lib/validators.ts` (primary candidate) **or** a schema file inside `frontend/src/components/business/StudentRegistrationForm/`
- **Action**: Update the student registration validation schema’s `firstName` rule
- **Details**:
  - Find the Zod v4 schema used for the form submission/validation.
  - Update `firstName` to: `.string().trim().min(1, "...").max(6, "...")`
  - Ensure trimming happens before min/max.
  - Keep error messages user-friendly; any valid copy acceptable.
- **Rationale**: Provides immediate client-side feedback and prevents submitting invalid values even if `maxLength` is bypassed.
- **Addresses**: Acceptance criteria #1, #3, #4, #5.

**Step 4: Add maxLength constraint to the firstName input in the form**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**` (the component file that renders the firstName field; likely `StudentRegistrationForm.tsx` or similar)
- **Action**: Update the firstName input props
- **Details**:
  - If the form uses the shared `TextInput` wrapper:
    - Prefer `maxLength={6}` if supported directly by the wrapper props, otherwise pass `inputProps={{ maxLength: 6 }}` down to MUI TextField.
  - Ensure this is applied only to `firstName` (not other fields).
  - Do not silently truncate on submit; this is just a typing guardrail. Validation still must reject >6.
- **Rationale**: Improves UX by preventing extra typing and aligning with requirement “should not accept more than 6 letters/characters”.
- **Addresses**: Acceptance criteria #1.

**Step 5: Ensure trimming behavior is applied consistently in frontend submission flow**
- **File**: `frontend/src/components/business/StudentRegistrationForm/**` (where submit handler maps form values to API payload)
- **Action**: Confirm the value sent is the trimmed value (or that schema parsing returns trimmed output)
- **Details**:
  - If the form uses `schema.parse(...)` / `safeParse(...)`, rely on Zod’s `.trim()` to produce the trimmed string in the parsed output.
  - Ensure the submit handler uses the parsed data (not raw uncontrolled input state) when calling the API.
- **Rationale**: Guarantees `" John "` becomes `"John"` before API call, matching backend behavior and acceptance criteria.
- **Addresses**: Acceptance criteria #3, #5.

**Step 6: Verify TextInput wrapper supports passing maxLength to the underlying input**
- **File**: `frontend/src/components/ui/TextInput/**` (likely `TextInput.tsx`)
- **Action**: If needed, extend the wrapper to accept `inputProps` or `maxLength`
- **Details**:
  - Inspect current prop types and how props are forwarded to MUI TextField.
  - If `inputProps` is not currently forwarded, add support so the form can set `maxLength`.
  - Keep exports consolidated; keep types exported via `export type`.
- **Rationale**: Ensures the form can actually enforce `maxLength` at the DOM level.
- **Addresses**: Acceptance criteria #1 (UI prevention).

### 4. VALIDATION STRATEGY

- **Backend (Zod v3)**: `z.string().trim().min(1).max(6)`
  - Trimming ensures whitespace-only becomes `""` and fails `.min(1)`.
  - `.max(6)` ensures any 7+ characters are rejected.
- **Frontend (Zod v4)**: same rule to ensure immediate feedback and prevent invalid submit.
- **Frontend input**: `maxLength=6` to prevent typing beyond 6 characters in normal usage, but not relied upon for correctness.

### 5. INTEGRATION POINTS

- Backend create-student endpoint is assumed to already validate requests using `StudentSchema`; updating the schema automatically enforces the rule across API usage.
- Frontend:
  - `frontend/src/app/students/new/page.tsx` likely renders `StudentRegistrationForm`; no change expected unless schema is defined there.
  - `StudentRegistrationForm` integrates:
    - UI layer (`TextInput`)
    - Validation layer (Zod schema)
    - API layer (`frontend/src/lib/api.ts`)

### 6. EDGE CASES & CONSIDERATIONS

- The requirement says “6 letters” but acceptance criteria says “6 characters”; implement **max 6 characters** (as specified).
- Names with internal spaces like `"Jo hn"` count as characters; trimming only removes leading/trailing whitespace.
- Ensure no silent truncation on submit: if a value >6 somehow exists (programmatic set), frontend schema should block submission and backend should reject.
- Keep error messages clear; consistency FE/BE preferred but not required.

### 7. IMPLEMENTATION CONSTRAINTS

- Focus only on implementing the max-length + trim validation across FE/BE and wiring `maxLength` in the UI.
- Do **not** add tests in this planning node (tests handled separately).
- Follow repo conventions:
  - Arrow functions only
  - Consolidated exports at end
  - Hyphenated filenames
  - `export type` separately
- Maintain backward compatibility except for the intended validation tightening (existing longer first names are out of scope).