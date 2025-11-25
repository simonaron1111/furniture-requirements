# Test Case Document — User Req 0002: Hardware Association Logic

## Requirement
The system must automatically add specific hardware to the BOM based on the components added to the furniture design (e.g., adding a door implies adding hinges; adding a drawer implies adding slides).

Scenario:
- Given the user has added a "Door" component to the front of the furniture box.
- When the BOM is generated.
- Then the list must include "Standard Hinges" and "Door Handle" with the correct quantities (e.g., 2 hinges per door up to 1000 mm height).

Acceptance criteria:
- Door adds ≥ 2 hinges (logic based on door height) and 1 handle.
- 1 Drawer adds 1 pair of drawer slides and 1 handle.
- Deleting the component from the 3D view removes the associated hardware from the BOM.

---

## Assumptions (explicit)
- Units and measurements are in millimetres (mm).
- Door height is measured from the Door component bounding box height (Hdoor) in mm.
- Default hinge-count policy (used by tests unless product defines other thresholds):
  - Hdoor ≤ 1000 mm → 2 hinges per door
  - 1000 mm < Hdoor ≤ 2000 mm → 3 hinges per door
  - Hdoor > 2000 mm → 4 hinges per door
  (Note: these thresholds are illustrative — product must confirm exact rules. Tests are parametric so thresholds can be substituted.)
- A “pair of slides” counts as one BOM line item with quantity = 1 pair (or equivalent two pieces depending on BOM representation).
- "Door Handle" and "Drawer Handle" are single items per door/drawer respectively.
- Hardware items are uniquely identified in BOM by SKU or canonical name (e.g., "STD-HINGE-01", "SLIDE-PAIR-450", "HANDLE-STD-25").
- BOM generation can be triggered automatically on model change or by explicit refresh; tests will account for both behaviors.
- Deleting a component should remove its associated hardware items only if those items are not still required by any remaining components (e.g., if two doors exist and one is deleted, hinge quantity reduces accordingly; if zero doors remain, hinge lines are removed).
- The association logic is deterministic and pure with respect to inputs (components list and their properties); any external lookups (e.g., SKU resolution) are mocked in component-level tests.

---

## Derived values and sample data (quick reference)
- Example Door: Hdoor = 900 mm → hinges = 2, handles = 1
- Example Door: Hdoor = 1500 mm → hinges = 3, handles = 1
- Example Drawer: adds slides = 1 pair, handle = 1

---

## Structured Test Cases

ID: TC-0002-001  
Title: Adding a single door adds correct hardware (basic happy path)  
Related Requirement: User Req - 0002  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Empty furniture box project or project with no doors/drawers.
- Auto-BOM generation enabled (or test will trigger BOM generation).

Test Data:
- Add one Door component to the front face with Hdoor = 900 mm.

Steps:
1. Add Door component (H = 900 mm) to the project and save.
2. Generate or refresh BOM.
3. Inspect BOM lines for door hardware.

Expected Result:
- BOM includes "Standard Hinges" with quantity = 2.
- BOM includes "Door Handle" with quantity = 1.
- Hardware line item names/SKUs are present and units clear (e.g., "2 pcs" or "2 hinges").
- No other door-related hardware lines are present.

Postconditions / Cleanup:
- Remove test Door component (or revert project).

---

ID: TC-0002-002  
Title: Door hinge count scales with door height (threshold behavior)  
Related Requirement: Door adds ≥ 2 hinges logic  
Priority: High  
Test Type: Functional / Numeric

Preconditions:
- Project ready; thresholds are as assumed or configured in the system.

Test Data:
- Door A: Hdoor = 1000 mm → expect 2 hinges
- Door B: Hdoor = 1500 mm → expect 3 hinges
- Door C: Hdoor = 2200 mm → expect 4 hinges

Steps:
1. Add Door A (1000 mm), Door B (1500 mm), Door C (2200 mm) to the project and save.
2. Generate/refresh BOM.
3. Inspect hinge quantities per door and aggregate totals.

Expected Result:
- Door A contributes 2 hinges; Door B contributes 3; Door C contributes 4.
- BOM shows either separate line items per door instance or aggregated totals:
  - If aggregated, total hinges = 2 + 3 + 4 = 9.
- Each door contributes exactly 1 handle; aggregated handle quantity = 3.

Postconditions:
- Remove test doors.

---

ID: TC-0002-003  
Title: Adding a single drawer adds slides pair and handle  
Related Requirement: Drawer adds 1 pair of slides and 1 handle  
Priority: High  
Test Type: Functional

Preconditions:
- Empty project or no drawers present.

Test Data:
- Add one Drawer component.

Steps:
1. Add Drawer component, save project.
2. Generate/refresh BOM.
3. Inspect BOM for drawer hardware.

Expected Result:
- BOM contains "Drawer Slides (pair)" with quantity = 1.
- BOM contains "Drawer Handle" with quantity = 1.
- Slides are represented as one pair (or equivalent two pieces if BOM schema requires piece-level reporting).

Postconditions:
- Remove test drawer.

---

ID: TC-0002-004  
Title: Deleting a component removes associated hardware from BOM  
Related Requirement: Deleting component removes associated hardware  
Priority: High  
Test Type: Integration / End-to-end

Preconditions:
- Project with two identical doors (each Hdoor = 900 mm) present.

Test Data:
- Start with 2 doors (expected hinges = 4 total, handles = 2).

Steps:
1. Generate BOM and verify hinge quantity = 4, handle quantity = 2.
2. Delete one Door component and save the model.
3. Refresh BOM.

Expected Result:
- BOM now shows hinges = 2 and handles = 1 (reflecting remaining single door).
- No orphaned hardware lines remain for the deleted door.
- If BOM initially aggregated per SKU, counts decrement accordingly.

Postconditions:
- Re-add or revert deletion if necessary.

---

ID: TC-0002-005  
Title: Multiple doors aggregate hardware correctly and avoid duplicate SKU lines  
Related Requirement: Functional completeness / deduplication  
Priority: Medium  
Test Type: Functional

Preconditions:
- Project supports multiple door components.

Test Data:
- Add three doors (H = 900 mm each).

Steps:
1. Add three doors, save model.
2. Generate BOM.

Expected Result:
- BOM contains one "Standard Hinges" line with quantity = 6 (3 doors × 2 hinges).
- BOM contains one "Door Handle" line with quantity = 3.
- No duplicate line entries per same SKU; quantities aggregated unless configuration dictates per-instance lines.

Postconditions:
- Remove test doors.

---

ID: TC-0002-006  
Title: Resizing a door updates hinge count on BOM (recalculation)  
Related Requirement: BOM updates when model changes  
Priority: High  
Test Type: Integration / End-to-end

Preconditions:
- Project contains a Door with Hdoor = 900 mm (2 hinges initially).

Test Data:
- Resize door to Hdoor = 1500 mm.

Steps:
1. Resize the Door component to 1500 mm in 3D view and save.
2. Refresh BOM.

Expected Result:
- Hinges for that door update from 2 to 3.
- Aggregate hinge count and handle count update accordingly.
- No stale counts remain.

Postconditions:
- Revert door size.

---

ID: TC-0002-007  
Title: Hardware not added when autopopulate hardware setting disabled (user preference)  
Related Requirement: System configuration / user preferences (implicit)  
Priority: Medium  
Test Type: Functional / Config

Preconditions:
- The system has a user/project setting "Auto-add hardware to BOM" (default ON). Tests assume the setting exists; if not, this test is skipped or marked N/A.

Test Data:
- Setting switched OFF.
- Add one Door component.

Steps:
1. Set "Auto-add hardware to BOM" = OFF (project or user preference).
2. Add Door component and save.
3. Generate/refresh BOM.

Expected Result:
- No auto-added hardware lines (no hinges or handle) appear.
- The BOM may include a note or flag indicating hardware auto-add is disabled.
- Re-enable setting for subsequent tests.

Postconditions:
- Re-enable auto-add setting.

---

ID: TC-0002-008  
Title: Invalid/partial component state does not produce hardware (defensive behavior)  
Related Requirement: Robustness / validation  
Priority: Medium  
Test Type: Negative

Preconditions:
- Door component created but missing essential dimension (height = null or 0) or marked invalid by model.

Test Data:
- Door with Hdoor = 0 or NaN.

Steps:
1. Add invalid Door to project and save.
2. Generate/refresh BOM.

Expected Result:
- System does not add hinge/handle lines for invalid door.
- BOM includes validation/warning entry describing invalid component.
- No hardware with NaN or zero dimensions listed.

Postconditions:
- Remove invalid component.

---

ID: TC-0002-009  
Title: Backend mapping: HardwareAssociationEngine returns deterministic hardware list (unit test)  
Related Requirement: Backend/component behavior (recommended)  
Priority: High  
Test Type: Unit / Component

Preconditions:
- HardwareAssociationEngine or equivalent backend function exists that maps components -> hardware items.
- External services (SKU service, DB) are mocked/stubbed.

Test Data:
- Input: components = [{type: "Door", height: 900, id: "door-1"}]

Steps:
1. Call hardwareList = HardwareAssociationEngine.mapComponentsToHardware(components).
2. Inspect returned hardwareList objects.

Expected Result:
- hardwareList contains entries:
  - {sku: "STD-HINGE-01", name: "Standard Hinges", quantity: 2}
  - {sku: "HANDLE-STD-25", name: "Door Handle", quantity: 1}
- The function did not call any repository or DB (assert mocks not invoked) if it is intended to be pure.
- Deterministic inputs produce deterministic outputs.

Postconditions:
- None.

---

ID: TC-0002-010  
Title: Internationalization and unit formatting for hardware quantities and labels  
Related Requirement: UI / BOM formatting (optional)  
Priority: Low  
Test Type: UI / Localization

Preconditions:
- BOM UI supports locales with different number formatting.

Steps:
1. Set UI locale to one using comma decimal separators (e.g., fr-FR).
2. Add door(s) and generate BOM.

Expected Result:
- Quantities are integers and shown correctly (no decimalization).
- Labels and unit text for hardware are localized if translations exist.
- No confusion between "pair" vs numeric quantity due to locale formatting.

Postconditions:
- Reset locale.

---

## Gherkin / BDD Scenarios

Feature: Automatic Hardware Association for Components
  In order to produce a correct BOM that includes required hardware
  As a furniture designer using the system
  I want hinges, handles, and slides to be added/removed automatically when I add or remove doors/drawers

Background:
  Given the BOM generation logic is enabled
  And hinge thresholds are configured as: ≤1000mm:2, 1001–2000mm:3, >2000mm:4 (test assumption)

Scenario: Adding a single door adds 2 hinges and 1 handle
  Given I added a Door of height 900 mm to the front face
  When the BOM is generated
  Then the BOM shall include "Standard Hinges" with quantity 2
  And the BOM shall include "Door Handle" with quantity 1

Scenario: Deleting a door removes its hardware
  Given a project has two doors with height 900 mm
  And the BOM shows hinges = 4 and handles = 2
  When I delete one door and refresh BOM
  Then the BOM shall show hinges = 2 and handles = 1

Scenario: Adding a drawer adds 1 pair of slides and 1 handle
  Given I added a Drawer to the project
  When the BOM is generated
  Then the BOM shall include "Drawer Slides (pair)" with quantity 1
  And the BOM shall include "Drawer Handle" with quantity 1

Scenario: Resizing door changes hinge count
  Given I have a Door with height 900 mm and BOM shows 2 hinges
  When I change the door height to 1500 mm and refresh BOM
  Then the BOM shall show 3 hinges for that door

Scenario: Auto-add disabled prevents hardware items from being added
  Given the "Auto-add hardware to BOM" setting is OFF
  When I add a Door and generate BOM
  Then the BOM shall not include hinges or handles for the added door

---

## Mapping to Acceptance Criteria
- "Door adds ≥ 2 hinges (logic based on door height) and 1 handle." — Covered by TC-0002-001, TC-0002-002, TC-0002-006.
- "1 Drawer adds 1 pair of drawer slides and 1 handle." — Covered by TC-0002-003.
- "Deleting the component from the 3D view removes the associated hardware from the BOM." — Covered by TC-0002-004.

---

## Edge Cases and Additional Recommendations
- Confirm and document exact hinge-threshold rules with product/engineering; make tests parametric so thresholds can be updated without rewriting tests.
- Test mixed cases: doors of different heights on same project and ensure aggregate quantities match sum of per-door requirements.
- Test doors with split panels (double doors) where handles/hinges rules may differ.
- Test hardware SKU resolution: ensure mapping between logical hardware names and SKUs is stable and is applied at BOM display time; if SKU lookup requires DB, tests should mock the lookup and validate the mapping separately.
- Concurrency: test that two clients editing the model concurrently produce deterministic final hardware counts (e.g., merge operations or last-write-wins behavior).
- Permissions: verify only authorized users can change components or trigger BOM changes if relevant.
- Performance: test BOM generation time when a large number of components are present (e.g., >200 doors/drawers).
- Regression automation: add unit tests for HardwareAssociationEngine logic and E2E tests that assert BOM invariants (no orphan hardware, counts consistent with components).

---

## Implementation / Test Automation Guidance
- Automate UI E2E tests using a framework such as Cypress, Playwright, or Selenium:
  - Actions: add/remove/resize component in 3D/2D view (via UI or API), trigger BOM refresh, assert BOM table rows and quantities.
- For backend/component tests, use unit test frameworks (JUnit, pytest, xUnit):
  - Inject/mock any external services (DB, SKU service) and assert they are not called when logic is supposed to be pure.
  - Tests should call the mapping function with well-formed component objects and assert deterministic returned hardware lists.
- Use test data builders to easily create Door and Drawer objects with specified heights/depths/metadata.
- For DB-backed BOM representations, use an isolated test database or transactional test fixtures to avoid polluting shared state.

---

## Test Data Summary (quick reference)
- Door_small: Hdoor = 900 mm → Hinges=2, Handle=1
- Door_medium: Hdoor = 1500 mm → Hinges=3, Handle=1
- Door_tall: Hdoor = 2200 mm → Hinges=4, Handle=1
- Drawer: Slides(pair)=1, Handle=1

---

## What I produced and next
I created a single, ready-to-run Markdown test-case document for User Req 0002 that includes assumptions, a clear set of structured test cases (happy path, negative, integration and unit-level), corresponding Gherkin scenarios, mapping to acceptance criteria, edge-case recommendations, and automation guidance. If you’d like, I can now (pick one): export this to CSV for your test management tool, generate Cypress/Playwright E2E scripts for the UI flows, or provide unit-test skeletons for your backend HardwareAssociationEngine in your preferred language/framework.