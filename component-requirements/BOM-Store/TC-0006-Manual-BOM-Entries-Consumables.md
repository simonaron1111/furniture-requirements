# Test Case Document — User Req 0006: Manual BOM Entries (Consumables)

## Requirement
The user must be able to manually add items to the generated BOM for consumables that are not geometrically drawn (e.g., Glue, Dowels, Feet).

Scenario:
- Given the automatically generated BOM is displayed.
- When the user clicks "Add Custom Item", fills in "Wood Glue", Quantity "1", and Price "5.00".
- Then this item is persisted in the database linked to this specific project and included in the Grand Total.

Acceptance criteria:
- Manual entries are visually distinct from auto-generated entries (e.g., different icon or highlight).
- Manual entries persist even if the 3D model is resized and the BOM auto-recalculates.

---

## Assumptions (explicit)
- Manual items are created via a UI flow ("Add Custom Item" button) that opens a modal or inline form.
- Required fields for a manual BOM line: name (string), quantity (integer), unitPrice (decimal), unit (optional, e.g., "ea" or "pair"), category (optional, e.g., "consumable"), optional notes.
- Manual entries are stored in a bom_custom_items table or in the bom_line_items table with a flag is_manual = true, and a project_id foreign key.
- Manual items have a unique id and metadata including created_by, created_at, and project_id.
- Manual entries are visually distinguished in UI by icon, color, or label ("manual") and sortable/filterable.
- The BOM rendering logic merges auto-generated BOM lines and manual items into a single view while preserving manual flags and ensuring manual items are not overwritten by recalculation. Recalculation updates only auto-generated lines unless user explicitly chooses to "recalculate and clear manual items".
- Currency and locale conventions follow the project's existing configuration; prices are stored as decimals and displayed with local formatting.
- Permissions: only users with edit permission for the project can add/edit/delete manual BOM items.
- Deleting a manual item requires confirmation and removes it from DB and Grand Total.
- Concurrency: multiple users may edit BOM simultaneously; last write or merge strategy applies (tests document behavior).

---

## Derived example (for tests)
- User action: Click "Add Custom Item" -> enter:
  - name: "Wood Glue"
  - quantity: 1
  - unitPrice: 5.00 (assume currency USD)
  - unit: "bottle"
  - note: "1 bottle for glue"
- Expected DB row:
  - { id: "...", project_id: 101, name: "Wood Glue", quantity: 1, unit_price: 5.00, is_manual: true, created_by: "userX", created_at: ... }

---

## Test Types & Scope
- UI E2E: simulate user adding a custom item, visual distinction, inclusion in Grand Total, persistence across reloads and model changes.
- Backend unit: ManualBOMService.createCustomItem(), update/delete, DB persistence and transactional integrity.
- Integration: Ensure manual items survive auto-recalc, verify merging rules (manual vs auto).
- Security/permissions: only authorized users can add/edit/delete manual items.
- Negative/edge: invalid inputs, missing required fields, negative quantities, extremely large price values, concurrent edits.
- Localization: currency format and decimal separators displayed per locale.
- Auditability: created_by and timestamps persisted and shown on hover or details.

---

## Structured Test Cases

ID: TC-0006-001  
Title: Add custom item via UI and persist to DB (happy path)  
Related Requirement: User Req - 0006  
Priority: High  
Test Type: E2E / Acceptance

Preconditions:
- Project 101 exists, user "userX" is logged in with edit permission.
- BOM initially generated and visible.
- UI shows "Add Custom Item" control.

Test Data:
- name = "Wood Glue"
- quantity = 1
- unitPrice = 5.00 (currency USD)
- unit = "bottle"

Steps:
1. Open project 101 and navigate to BOM.
2. Click "Add Custom Item".
3. Fill fields: name, quantity, unitPrice, unit, optional note.
4. Submit/save the custom item.
5. Wait for UI confirmation and for the BOM to refresh/display the new line.
6. Inspect DB row for bom_custom_items or bom_line_items where is_manual = true and project_id = 101.

Expected Result:
- New manual BOM line appears in BOM list immediately.
- The line is visually distinct (e.g., manual icon, different background or "Manual" label).
- DB contains a persisted entry with correct fields and created_by = userX, linked to project 101.
- The Grand Total updates to include the custom item (adds $5.00).
- A success message or undo option is available.

Postconditions / Cleanup:
- Delete the test manual item (or rollback test transaction).

---

ID: TC-0006-002  
Title: Manual entry persists after 3D model resize and auto-recalc  
Related Requirement: Manual entries persist across auto recalculation  
Priority: High  
Test Type: Integration / E2E

Preconditions:
- Manual item "Wood Glue" exists for project 101.
- Auto-BOM recalculation triggers when 3D model is changed.

Steps:
1. Resize 3D model (change dimensions) and save project (trigger automatic BOM recalculation).
2. Refresh BOM view (or wait for auto-update).
3. Inspect BOM list and Grand Total.

Expected Result:
- Manual "Wood Glue" item remains present and unchanged after recalculation.
- Grand Total reflects both recalculated auto-lines and the manual item.
- The manual item retains its is_manual flag and created_by metadata.

Postconditions:
- Revert model size or remove test manual item.

---

ID: TC-0006-003  
Title: Manual entry visually distinct from auto-generated entries  
Related Requirement: Visual differentiation acceptance criterion  
Priority: Medium  
Test Type: UI / Accessibility

Preconditions:
- BOM contains auto lines and at least one manual item.

Steps:
1. Open BOM and visually inspect manual vs auto items.
2. Verify visual markers (icon, color, badge) and accessible text (ARIA label or tooltip).

Expected Result:
- Manual items have a consistent visual marker and accessible label "Manual entry".
- Color/contrast meets accessibility standards (AA).
- Sorting/filtering by manual vs auto works (if UI supports).

Postconditions:
- None.

---

ID: TC-0006-004  
Title: Validation rejects invalid manual input (negative qty or missing name/price)  
Related Requirement: Input validation / data integrity  
Priority: High  
Test Type: UI / Unit

Preconditions:
- BOM UI open with "Add Custom Item" form.

Test Data:
- Case A: quantity = -1
- Case B: missing name
- Case C: unitPrice = -5.00
- Case D: extremely large unitPrice or quantity exceeding business limits

Steps:
1. For each case, attempt to submit the Add Custom Item form.
2. Observe validation errors and form blocking behavior.

Expected Result:
- Form prevents submission and shows clear validation messages:
  - "Name is required"
  - "Quantity must be a positive integer"
  - "Price must be a non-negative number"
  - "Quantity/price exceeds maximum allowed" for limits
- No DB row is created.

Postconditions:
- None.

---

ID: TC-0006-005  
Title: Deleting a manual item removes it from DB and updates Grand Total  
Related Requirement: CRUD operations for manual entries  
Priority: Medium  
Test Type: E2E / Integration

Preconditions:
- Manual item exists in BOM for project 101.

Steps:
1. Click delete/remove on the manual item in BOM UI.
2. Confirm deletion in dialog.
3. Refresh BOM and check DB.

Expected Result:
- Item removed from BOM UI.
- DB row marked deleted or removed.
- Grand Total updates to subtract the item's cost.
- Audit log records deletion with user and timestamp.

Postconditions:
- None.

---

ID: TC-0006-006  
Title: Edit manual item updates DB and Grand Total (quantity or price change)  
Related Requirement: Edit behavior for manual items  
Priority: Medium  
Test Type: Integration

Preconditions:
- Manual item exists.

Steps:
1. Open edit dialog for the manual item.
2. Change quantity/price (e.g., quantity 2, price 5.00 -> 10.00).
3. Save changes.
4. Inspect DB and BOM UI.

Expected Result:
- DB row updated with new values and updated_by/updated_at recorded.
- BOM UI shows updated line cost and Grand Total recalculated accordingly.

Postconditions:
- Revert edits.

---

ID: TC-0006-007  
Title: Manual items included in export/print of BOM and in Grand Total export  
Related Requirement: Reporting/exports include manual entries  
Priority: Low / Optional  
Test Type: Integration / UI

Preconditions:
- Manual items present.

Steps:
1. Export BOM (CSV/PDF) or print.
2. Inspect exported data.

Expected Result:
- Manual items included in exported BOM with clear marker "Manual".
- Exported Grand Total includes manual items.
- Currency and formatting preserved.

Postconditions:
- None.

---

ID: TC-0006-008  
Title: Permission enforcement: unauthorized user cannot add/edit/delete manual items  
Related Requirement: Security / permissions  
Priority: High  
Test Type: Security / Integration

Preconditions:
- Two users: editor (has edit rights) and viewer (read-only).

Steps:
1. Log in as viewer and attempt to open Add Custom Item UI or invoke create API directly.
2. Attempt to edit or delete an existing manual item.

Expected Result:
- UI hides or disables Add/Edit/Delete controls for viewer.
- Backend API rejects unauthorized attempts with 403 Forbidden.
- No DB changes occur.

Postconditions:
- None.

---

ID: TC-0006-009  
Title: Concurrency: concurrent edits to manual item handled deterministically (last-write-wins or merge)  
Related Requirement: Concurrency / merging strategy  
Priority: Medium  
Test Type: Concurrency / Integration

Preconditions:
- Manual item exists and two users are editing simultaneously.

Steps:
1. User A opens edit form and changes quantity to 2 but does not save yet.
2. User B changes quantity to 3 and saves.
3. User A saves afterwards.

Expected Result:
- System uses defined conflict resolution:
  - If last-write-wins: final value = User A's saved value (2).
  - If merge or optimistic locking: User A receives conflict warning and must refresh or rebase changes.
- Tests assert the configured behavior.

Postconditions:
- None.

---

ID: TC-0006-010  
Title: Manual item creation via API is persisted and included in Grand Total (backend contract test)  
Related Requirement: API contract for programmatic creation  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Authenticated API client with edit permission.
- API endpoint: POST /api/projects/{projectId}/bom/manual-items

Test Data:
- POST payload: { name: "Wood Glue", quantity: 1, unitPrice: 5.00, unit: "bottle", note: "Test" }

Steps:
1. Call API to create manual item.
2. Read response and inspect DB.
3. Request BOM totals via GET /api/projects/{projectId}/bom-summary

Expected Result:
- API returns 201 Created with created item payload including id and is_manual = true.
- DB contains the created item linked to projectId.
- BOM summary GET shows Grand Total includes the 5.00 addition.
- Response includes created_by and timestamps.

Postconditions:
- Delete test item.

---

## Component / Unit Test Cases (ManualBOMService)

ID: MBS-CTC-0006-01  
Title: ManualBOMService.createCustomItem persists and returns DTO (unit)  
Priority: High  
Test Type: Unit

Preconditions:
- ManualBOMService with repository mocked (e.g., BomManualRepository).

Test Data:
- input DTO: { projectId: 101, name: "Wood Glue", quantity: 1, unitPrice: 5.00, unit: "bottle" }

Steps:
1. Call service.createCustomItem(inputDto, userId).
2. Mock repository.save to return persisted entity with id and timestamps.

Expected Result:
- Service returns DTO with id, is_manual = true, created_by = userId.
- Repository.save called once with entity mapped correctly (projectId, name, quantity, price).
- No interactions with auto-BOM generation component.

Postconditions:
- None.

---

ID: MBS-CTC-0006-02  
Title: ManualBOMService prevents duplicate manual items by same name (optional rule)  
Priority: Low / Configurable  
Test Type: Unit

Preconditions:
- Config: disallow duplicate manual names per project or allow duplicates.

Test Data:
- Existing manual item "Wood Glue"

Steps:
1. Call createCustomItem with same name.
2. Observe behavior based on config.

Expected Result:
- If duplicates disallowed: service returns validation error and repository.save not called.
- If allowed: service creates additional item.

Postconditions:
- None.

---

ID: MBS-CTC-0006-03  
Title: Manual items excluded from auto-recalc overwrite (is_manual protection)  
Priority: High  
Test Type: Integration

Preconditions:
- Auto-recalc process runs and creates/updates auto-generated bom_line_items.

Steps:
1. Ensure manual items exist alongside auto ones.
2. Trigger auto-recalc and merge process.
3. Inspect DB for manual and auto lines.

Expected Result:
- is_manual items are untouched by auto-recalc.
- Auto-recalc updates only lines where is_manual = false.
- No manual items are deleted unless user explicitly removes them.
- Merge logs or audit indicate protected manual items.

Postconditions:
- None.

---

ID: MBS-CTC-0006-04  
Title: Manual item persisted transactionally with BOM save (atomicity)  
Priority: Medium  
Test Type: Integration

Preconditions:
- Operation that saves model and manual items in one transaction supported.

Steps:
1. Submit request that updates 3D model and creates manual item together.
2. Force a failure (e.g., make repository throw after partial work) to validate rollback.

Expected Result:
- On failure, no partial state persists (either both model changes and manual item saved, or none saved).
- Transaction boundaries respected.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Manual BOM Entries (Consumables)
  In order to include non-geometric consumables in the BOM
  As a project editor
  I want to add, edit, and remove custom BOM items and have them persist and affect totals

Background:
  Given I am logged in as a project editor for project 101
  And the project has an auto-generated BOM

Scenario: Add a custom item and persist it
  Given I click "Add Custom Item"
  When I enter "Wood Glue", quantity "1", and price "5.00"
  And I save the item
  Then the BOM shall display a manual line "Wood Glue" with price "$5.00"
  And the item shall be stored in the database linked to project 101
  And the Grand Total shall include the $5.00

Scenario: Manual items remain after auto-recalculation
  Given a manual item "Wood Glue" exists in project 101
  When I resize the 3D model and BOM auto-recalculates
  Then "Wood Glue" shall still be present in the BOM and included in the Grand Total

Scenario: Manual entries are visually distinct
  Given the BOM contains manual and auto items
  When I view the BOM
  Then manual items shall be shown with a "Manual" badge/icon and accessible label

Scenario: Unauthorized user cannot create manual items
  Given I am a read-only user
  When I try to access "Add Custom Item"
  Then the control shall be hidden or disabled and API calls blocked with 403

Scenario: Missing required fields block creation
  Given I open Add Custom Item form
  When I submit without a name or price
  Then the system shall show validation messages and not create the item

---

## Mapping to Acceptance Criteria
- "Manual entries are visually distinct from auto-generated entries" — Covered by TC-0006-003, Gherkin scenarios.
- "Manual entries persist even if the 3D model is resized and the BOM auto-recalculates" — Covered by TC-0006-002, MBS-CTC-0006-03.

---

## Edge Cases and Recommendations
- Decide merge policy for manual vs auto lines: protect manual entries by marking them with is_manual; document behavior when auto-recalc would otherwise generate a line with same name/sku.
- Consider a "pin" or "lock" feature to protect manually added items from bulk operations (imports, syncs).
- Provide bulk-add and import functionality for consumables (CSV upload) and test import validation and deduplication.
- Provide an "undo" for manual item additions (temporary retention) to reduce accidental additions.
- Provide audit trail and export of manual items for procurement approvals.
- If pricing is required for Grand Total, ensure manual items include currency and unit price at creation; provide a UI to default currency from project settings.
- Offer a toggle in BOM settings: "Clear manual items on recalculation" (dangerous) — if present, add tests for that destructive action with confirmation dialogs.
- Consider a user preference to show/hide manual items in BOM or to collapse them into a consumables group.
- For multi-user scenarios, implement optimistic locking (version/timestamp) on manual items to prevent silent overwrite and add tests.

---

## Implementation / Test Automation Guidance
- UI E2E: use Playwright or Cypress to automate UI flow: open BOM, click Add Custom Item, fill form, save, assert UI list, assert Grand Total, and optionally call API to verify DB row.
- Backend unit tests: mock repository for ManualBOMService to assert persistence and validation behavior (pytest + unittest.mock, JUnit + Mockito).
- Integration tests: transactional test DB seeded with project rows, create manual item via API, assert persisted and included in BOM summary endpoint.
- API contract: POST /api/projects/{projectId}/bom/manual-items, PATCH /api/projects/{projectId}/bom/manual-items/{id}, DELETE /api/projects/{projectId}/bom/manual-items/{id}
  - Required fields: name, quantity, unitPrice, optional: unit, notes.
  - Responses include id, created_by, created_at, is_manual flag.
- Accessibility: ensure manual item badge has ARIA label; test with axe or accessibility scanner.
- Logging/telemetry: emit audit events on create/update/delete with user id and project id.
- Test data builders: create helper to insert and delete manual items in DB for tests.
- Use feature flags if this capability may be rolled out gradually; include tests for feature-flag off behavior (UI hidden).

---

## Test Data Summary (quick reference)
- Project: id = 101
- Test manual item:
  - name: "Wood Glue"
  - quantity: 1
  - unitPrice: 5.00
  - unit: "bottle"
  - expected DB: is_manual = true, project_id = 101

---

What I did: I wrote a single, ready-to-use Markdown test-case document for User Req - 0006 that captures UI, backend, integration and security tests, BDD scenarios, and implementation guidance so QA and engineering can implement automated tests and acceptance checks.

What's next: I can export this as a CSV for your test-management tool, generate API contract examples (OpenAPI fragment) and unit-test skeletons in your preferred language (e.g., pytest/JUnit), or create Playwright/Cypress E2E scripts that drive the "Add Custom Item" UI flow — tell me which output and target framework you want and I'll produce the code next.