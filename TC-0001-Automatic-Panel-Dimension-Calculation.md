# Test Case Document — User Req 0001: Automatic Panel Dimension Calculation

## Requirement
The system must automatically calculate the list of required wooden panels (cut list) based on the geometric dimensions of the furniture drawn in the 2D/3D view, accounting for material thickness.

Scenario:
- Given a furniture project exists with a defined outer "Box" of dimensions 2000 mm (H) × 1000 mm (W) × 600 mm (D) and a material thickness of 18 mm.
- When the user navigates to the "Bill of Materials" tab.
- Then the panel list should populate with specific parts (e.g., 2x Side Panels, 1x Top Panel, 1x Bottom Panel) showing calculated dimensions (e.g., Top Panel: 1000 mm × 600 mm).

Acceptance Criteria:
- The sum of internal panel dimensions + thickness must equal the total outer dimensions.
- Changes to the 3D model dimensions must trigger a recalculation of the BOM upon refresh.
- Dimensions must be displayed in millimetres (mm).

---

## Assumptions (explicit)
- Assembly rule used for these test cases:
  - Side panels are the vertical sides and are specified Height × Depth (H × D) and are listed as two pieces (left/right).
  - Top and Bottom panels are cut to fit between the two side panels. Therefore:
    - Top/Bottom Width = outer_width − 2 × thickness
    - Top/Bottom Depth = outer_depth
  - Thickness (t) is applied to each side where relevant (two sides reduce internal width by 2 × t).
- Thickness t = 18 mm unless the test changes it.
- Units displayed and tested are millimetres (mm).
- BOM lists cut-face dimensions for each panel and shows quantity and unit (mm).
- Decimal precision: decimals are allowed; equality checks should account for typical floating-point tolerances (e.g., ±0.1 mm if needed).

Derived numeric values used in tests (for outer box 2000 × 1000 × 600 mm and thickness 18 mm)
- internal_width = 1000 − 2 × 18 = 964 mm
- internal_height = 2000 − 2 × 18 = 1964 mm (if applicable)
- Verification sample: 964 + 2 × 18 = 1000 mm

---

## Structured Test Cases

ID: TC-0001
Title: BOM populates with expected panels and calculated dimensions (happy path)  
Related Requirement: User Req - 0001  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Project exists with outer box H=2000 mm, W=1000 mm, D=600 mm.
- Material thickness = 18 mm.
- User is logged in and can open the project.

Test Data:
- Outer H = 2000 mm, W = 1000 mm, D = 600 mm, thickness = 18 mm.

Steps:
1. Open the project and navigate to the "Bill of Materials" tab.
2. Observe the panel list populated by the system.

Expected Result:
- BOM lists at least: 2x Side Panels, 1x Top Panel, 1x Bottom Panel.
- Side Panels (2x): Height × Depth = 2000 mm × 600 mm (thickness = 18 mm shown separately if the UI lists it).
- Top Panel (1x): Width × Depth = 964 mm × 600 mm (964 = 1000 − 2×18).
- Bottom Panel (1x): Width × Depth = 964 mm × 600 mm.
- All dimension values are displayed in mm.
- Verification: internal_width (964 mm) + 2 × thickness (36 mm) = outer_width (1000 mm).

Postconditions / Cleanup:
- None (read-only check).

---

ID: TC-0002
Title: BOM recalculates when model dimensions change (refresh)  
Related Requirement: Recalculation acceptance criterion  
Priority: High  
Test Type: Integration / Acceptance

Preconditions:
- Use project from TC-0001.

Test Data:
- Change outer width from 1000 mm to 1200 mm (H and D unchanged), thickness = 18 mm.

Steps:
1. In 2D/3D view, update outer width to 1200 mm and save the model.
2. Navigate to Bill of Materials and refresh (or re-open BOM).

Expected Result:
- BOM updates to reflect new dimensions.
- internal_width = 1200 − 2 × 18 = 1164 mm.
- Top and Bottom Panel sizes update to 1164 mm × 600 mm.
- internal_width (1164) + 2 × thickness (36) = 1200 mm.

Postconditions:
- Revert width to 1000 mm if necessary.

---

ID: TC-0003
Title: Dimensions displayed in millimetres and formatted consistently  
Related Requirement: Dimensions must be displayed in mm  
Priority: Medium  
Test Type: UI / Functional

Preconditions:
- BOM page accessible.

Steps:
1. Navigate to BOM.
2. Inspect all numeric dimension fields and unit labels.

Expected Result:
- All dimensions show unit "mm" or the BOM unit setting is explicitly mm.
- Numeric values match computed mm values (e.g., 964 mm shown, not 0.964 m).
- Decimal precision is consistent across BOM (define expected decimal places in test run: e.g., 0 or 1 decimal place).

Postconditions:
- None.

---

ID: TC-0004
Title: Internal dimensions plus thickness equal outer dimensions (verification rule)  
Related Requirement: “The sum of internal panel dimensions + thickness must equal the total outer dimensions”  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Use project from TC-0001.

Steps:
1. From BOM, record internal_width (width of top/bottom panels).
2. Compute sum = internal_width + 2 × thickness.

Expected Result:
- sum equals outer_width exactly (964 + 36 = 1000 mm).
- If system reduces height by top and bottom panel thickness, run same check for height: internal_height + 2 × thickness = outer_height.

Postconditions:
- None.

---

ID: TC-0005
Title: Recalculation triggered when updating 3D model (change in 3D view)  
Related Requirement: Changes to the 3D model dimensions must trigger a recalculation of the BOM upon refresh  
Priority: High  
Test Type: Integration / End-to-end

Preconditions:
- Access to 3D view with edit capability.

Test Data:
- Change depth D from 600 mm to 400 mm; thickness = 18 mm.

Steps:
1. In 3D view change depth to 400 mm and save the model.
2. Switch to BOM tab and refresh.

Expected Result:
- Panel depths update to 400 mm for panels whose depth equals outer_depth.
- Top/Bottom/Side panels depth fields show 400 mm.
- No stale data shown on BOM after refresh.

Postconditions:
- Revert depth to 600 mm if needed.

---

ID: TC-0006
Title: Reject or warn on invalid material thickness that would produce non-positive internal dimensions  
Related Requirement: Robustness / validation (implicit)  
Priority: High  
Test Type: Negative / Validation

Preconditions:
- Project with small outer dimension (for example outer_width = 30 mm).

Test Data:
- outer_width = 30 mm, thickness = 20 mm.

Steps:
1. Set thickness to 20 mm for the project and save.
2. Navigate to BOM or trigger panel calculation.

Expected Result:
- System detects internal_width = 30 − 2 × 20 = −10 mm (invalid).
- BOM displays a clear validation/warning message (e.g., "Material thickness too large — no valid internal width") or prevents calculation.
- No negative panel sizes are listed.

Postconditions:
- Reset thickness to valid value.

---

ID: TC-0007
Title: Support for non-integer thickness values and precision check  
Related Requirement: Units/precision  
Priority: Medium  
Test Type: Functional / Numeric precision

Preconditions:
- Project as TC-0001.

Test Data:
- thickness = 12.5 mm.

Steps:
1. Set thickness to 12.5 mm and refresh BOM.

Expected Result:
- internal_width = 1000 − 2 × 12.5 = 975.0 mm.
- Top/Bottom panel widths show 975.0 mm (or 975 mm) in mm units.
- internal_width + 2 × thickness = 975.0 + 25.0 = 1000.0 mm (verify within acceptable tolerance).

Postconditions:
- Reset thickness to 18 mm.

---

ID: TC-0008
Title: BOM contains expected counts and no duplicates or missing panels  
Related Requirement: Functional completeness  
Priority: Medium  
Test Type: Functional

Preconditions:
- Project with default box.

Steps:
1. Open BOM.
2. Verify each expected panel appears and has correct quantity.

Expected Result:
- Exactly 2 side panels, exactly 1 top, exactly 1 bottom (unless model requires otherwise).
- No duplicate rows for the same panel unless appropriate; quantity field reflects the number of pieces.

Postconditions:
- None.

---

ID: TC-0009
Title: Persistence: saved project retains calculated BOM values (load/save)  
Related Requirement: Data persistence  
Priority: Medium  
Test Type: Integration

Preconditions:
- Project saved after calculation.

Steps:
1. Calculate BOM and save project.
2. Close project and re-open.
3. Navigate to BOM.

Expected Result:
- BOM values remain identical to saved values (no recompute drift).

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Automatic Panel Dimension Calculation  
  In order to produce a correct cut list  
  As a user of the furniture design tool  
  I want the BOM to show panel counts and calculated dimensions (in mm) that assemble to the outer box dimensions

Background:
  Given a project with outer box H=2000 mm, W=1000 mm, D=600 mm and material thickness=18 mm

Scenario: BOM shows expected panels and dimensions (happy path)  
  Given I open the "Bill of Materials" tab  
  When the system calculates panel dimensions  
  Then the BOM shall list 2 side panels with size 2000 mm × 600 mm  
  And the BOM shall list 1 top panel with size 964 mm × 600 mm  
  And the BOM shall list 1 bottom panel with size 964 mm × 600 mm  
  And internal_width + 2 × thickness = outer_width (964 + 36 = 1000 mm)

Scenario: Updating model dimensions triggers BOM recalculation  
  Given I change outer width to 1200 mm in the 3D view and save the model  
  When I refresh the "Bill of Materials" tab  
  Then the top and bottom panel widths shall update to 1164 mm (1200 − 2 × 18)  
  And internal_width + 2 × thickness = outer_width (1164 + 36 = 1200 mm)

Scenario: BOM displays values in millimetres  
  Given the BOM is visible  
  Then every dimension value shall include units "mm" or the BOM unit setting must be mm  
  And numeric values are displayed as millimetres (not meters)

Scenario: Invalid thickness preventing valid cut sizes  
  Given I set thickness = 20 mm for a project with outer width = 30 mm  
  When the system attempts to compute panel sizes  
  Then the system shall show a validation error explaining that material thickness is too large to produce valid panel dimensions

---

## Mapping to Acceptance Criteria
- Sum of internal panel dimensions + thickness equals outer dimensions: TC-0001, TC-0004.
- Changes to 3D model dimensions trigger recalculation on refresh: TC-0002, TC-0005.
- Dimensions displayed in millimetres: TC-0003, TC-0007.

---

## Notes, optional tests and edge cases (recommendations)
- Extend tests to back panel, shelves, drawer panels if the project includes them; verify formula per part.
- Include tests for different join types (butt, dado, rabbet) if supported — join type changes geometry formulas.
- Performance: measure BOM recalculation time on complex models and add thresholds (e.g., < 2s for simple boxes).
- Localization: ensure mm is preserved and number formatting (decimal separator) does not break numeric parsing.
- Accessibility: BOM must be navigable by keyboard and readable by screen readers.
- Security: ensure only authorized users can view or recalc BOM.
- Regression automation: add automated checks that verify the arithmetic invariant (internal + 2×thickness = outer) to catch formula regressions.

---

## Test Data Summary (quick reference)
- Base project: H=2000 mm, W=1000 mm, D=600 mm, thickness=18 mm.
- Derived sample values:
  - internal_width = 964 mm
  - internal_height = 1964 mm (if applicable)
  - internal_width + 2×thickness = 1000 mm
