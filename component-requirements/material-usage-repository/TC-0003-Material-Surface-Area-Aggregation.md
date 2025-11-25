# Test Case Document — User Req 0003: Material Surface Area Aggregation

## Requirement
The system must aggregate the total square meterage (m^2) required for each specific material type (e.g., White Melamine vs. Oak Veneer) to assist in purchasing sheets.

Scenario:
- Given the design uses "White Melamine" for the internal carcass and "Oak Veneer" for the door fronts.
- When the user views the BOM Summary section.
- Then the system displays two distinct totals: Total Area for White Melamine and Total Area for Oak Veneer.

Acceptance criteria:
- Area is calculated as Length × Width for every panel.
- Items with different material_id are not summed together.
- The total is rounded to two decimal places.

---

## Assumptions (explicit)
- Panel geometry is provided as width and length for the exposed/cut face used to purchase sheet material.
- Dimensions coming from model may be in millimetres (mm) or metres (m). The aggregator must convert to metres before area calculation when needed.
  - Conversion: metres = millimetres / 1000. Area (m^2) = (length_m × width_m).
- Thickness does not affect surface area for sheet purchasing and must be ignored in area calculation.
- Material identity is determined by material_id (e.g., "MAT-WHITE-MEL", "MAT-OAK-VNR"); different material_id values must not be aggregated together.
- Rounding mode: round to two decimal places using standard rounding (round half up) unless otherwise specified by product policy.
- Final totals displayed in BOM Summary must show unit "m^2" and numeric value with two decimals (e.g., "6.25 m^2").
- Tolerance for floating arithmetic: internal calculations may use higher precision, but final reported value must be rounded to 2 decimals.

---

## Derived formulas / examples
- Given panel dims in mm: L = 1000 mm, W = 600 mm
  - L_m = 1000 / 1000 = 1.0 m
  - W_m = 600 / 1000 = 0.6 m
  - Area = 1.0 × 0.6 = 0.6 m^2
  - Rounded to 2 decimals -> 0.60 m^2
- Aggregation example:
  - White Melamine panels areas: 0.6 + 0.96 + 1.50 = 3.06 m^2 -> display 3.06 m^2

---

## Test Types and Scope
- UI acceptance tests: verify BOM Summary displays correct per-material totals.
- Unit/component tests: MaterialAreaAggregator service pure function tests (grouping, conversion, rounding).
- Integration tests: verify updates (add/remove/resize panels) update summary on refresh.
- Negative/edge tests: missing dimensions, zero-area panels, mixed units, extremely small/large dimensions, rounding edge cases.
- Performance test (optional): aggregate a large number of panels and measure latency.

---

## Structured Test Cases

ID: TC-0003-001  
Title: Aggregate area for two distinct materials (happy path)  
Related Requirement: User Req - 0003  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Project contains panels:
  - 4 internal carcass panels (material_id = "MAT-WHITE-MEL"):
    - Panel A: 1000 mm × 600 mm
    - Panel B: 1000 mm × 600 mm
    - Panel C: 1000 mm × 600 mm
    - Panel D: 1000 mm × 600 mm
  - 2 door fronts (material_id = "MAT-OAK-VNR"):
    - Door E: 1000 mm × 500 mm
    - Door F: 1000 mm × 500 mm

Test Data (explicit):
- White Melamine panels: each area = (1.0 × 0.6) = 0.60 m^2 → total = 4 × 0.60 = 2.40 m^2
- Oak Veneer doors: each area = (1.0 × 0.5) = 0.50 m^2 → total = 2 × 0.50 = 1.00 m^2

Steps:
1. Ensure panels exist with configured material_id and dimensions in mm.
2. Open BOM Summary.

Expected Result:
- BOM Summary displays:
  - White Melamine: 2.40 m^2
  - Oak Veneer: 1.00 m^2
- Each value shown with two decimals and unit "m^2".

Postconditions / Cleanup:
- None.

---

ID: TC-0003-002  
Title: Items with different material_id are not summed together  
Related Requirement: Acceptance criterion (separate totals by material_id)  
Priority: High  
Test Type: Unit / Functional

Preconditions:
- Two panels with identical dimensions but different material_id:
  - Panel A: 1000 × 600 mm, material_id = "MAT-WHITE-MEL"
  - Panel B: 1000 × 600 mm, material_id = "MAT-WHITE-MEL-ALT"

Steps:
1. Run MaterialAreaAggregator on the panel list.
2. Inspect aggregated totals.

Expected Result:
- Two separate totals shown:
  - MAT-WHITE-MEL: 0.60 m^2
  - MAT-WHITE-MEL-ALT: 0.60 m^2
- No mixing/aggregation across different material_id.

Postconditions:
- None.

---

ID: TC-0003-003  
Title: Rounding totals to two decimal places (round-half-up behavior)  
Related Requirement: Totals rounded to 2 decimals  
Priority: High  
Test Type: Unit / Numeric

Preconditions:
- Panels produce total area with more than 2 decimals, e.g.:
  - Panel 1: 333 mm × 333 mm = (0.333 × 0.333) = 0.110889 m^2
  - Panel 2: 333 mm × 333 mm = 0.110889 m^2
  - Total raw = 0.221778 m^2

Steps:
1. Aggregate these two panels for material "MAT-XYZ".
2. Apply rounding rule to 2 decimals.

Expected Result:
- Raw total = 0.221778 → rounded (2 decimals) = 0.22 m^2 (assuming round-half-up).
- Display value: 0.22 m^2.

Postconditions:
- None.

---

ID: TC-0003-004  
Title: Unit conversion when input dimensions are in metres vs millimetres  
Related Requirement: Correct unit handling and area calc  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Panel A: dimensions provided as mm: 1000 × 600 mm (area 0.60 m^2)
- Panel B: dimensions provided as m: 1.2 × 0.8 m (area 0.96 m^2)
- Both panels same material_id.

Steps:
1. Aggregate panels which have mixed unit representations (explicit unit indicator present).
2. Verify conversion and aggregation.

Expected Result:
- Aggregator converts mm to m for Panel A: 1.0 × 0.6 = 0.60
- Panel B already in metres: 1.2 × 0.8 = 0.96
- Total = 0.60 + 0.96 = 1.56 → display 1.56 m^2

Postconditions:
- None.

---

ID: TC-0003-005  
Title: Deleting a panel updates the material total in BOM Summary (dynamic update)  
Related Requirement: BOM updates on model change  
Priority: High  
Test Type: Integration / End-to-end

Preconditions:
- Project initially has two White Melamine panels each 1.0 × 0.6 m (total 1.20 m^2).

Steps:
1. Verify BOM Summary shows White Melamine = 1.20 m^2.
2. Delete one White Melamine panel and save model.
3. Refresh BOM Summary.

Expected Result:
- White Melamine total updates to 0.60 m^2.
- No leftover/ghost entries.

Postconditions:
- Re-add removed panel if needed.

---

ID: TC-0003-006  
Title: Zero or missing dimension panels are ignored and logged (defensive behavior)  
Related Requirement: Robustness / validation  
Priority: Medium  
Test Type: Negative / Unit

Preconditions:
- Panel with missing length or width (null) or zero dimension exists.

Steps:
1. Run aggregator with a mix of valid and invalid panels.
2. Inspect result and logs.

Expected Result:
- Invalid panels are excluded from aggregation.
- Aggregator returns totals computed only from valid panels.
- A warning or validation message returned or logged that identifies skipped panel(s) and reason (e.g., "Panel id=xyz missing width").

Postconditions:
- None.

---

ID: TC-0003-007  
Title: Floating-point accumulation and rounding stability (many small panels)  
Related Requirement: Numeric stability / rounding aggregate after sum  
Priority: Medium  
Test Type: Unit / Numeric

Preconditions:
- 100 panels each 333 mm × 333 mm for same material:
  - each = 0.110889 m^2, raw total = 11.0889 m^2

Steps:
1. Aggregate 100 such panels.
2. Verify raw sum and final rounded total.

Expected Result:
- Raw total ≈ 11.0889 → rounded = 11.09 m^2
- Ensure accumulation algorithm avoids excessive rounding error (use double precision, sum then round).

Postconditions:
- None.

---

ID: TC-0003-008  
Title: Performance sanity test for large BOM (optional)  
Related Requirement: Performance / responsiveness  
Priority: Low / Optional  
Test Type: Performance / Integration

Preconditions:
- Large model with >1000 panels across multiple materials.

Steps:
1. Run area aggregation routine and measure execution time.
2. Verify memory usage and responsiveness.

Expected Result:
- Aggregation completes within acceptable threshold (define threshold for your environment, e.g., < 500 ms).
- Results correct and rounded to 2 decimals.

Postconditions:
- None.

---

## Component / Unit Test Cases (MaterialAreaAggregator)

ID: MAT-CTC-0001  
Title: MaterialAreaAggregator returns map material_id -> total_m2 (pure function)  
Priority: High  
Test Type: Unit

Test Data:
- panels = [
    {id: "p1", material_id: "MAT-WHITE-MEL", length_mm: 1000, width_mm: 600},
    {id: "p2", material_id: "MAT-OAK-VNR", length_mm: 1000, width_mm: 500}
  ]

Steps:
1. Call aggregator.aggregate(panels).
2. Inspect returned Map.

Expected Result:
- { "MAT-WHITE-MEL": 0.60, "MAT-OAK-VNR": 0.50 } (values rounded to 2 decimals)

Additional checks:
- aggregator must not call DB or I/O; mock injected dependencies should be asserted not-called.

---

ID: MAT-CTC-0002  
Title: Aggregator handles fractional dimensions and rounding correctness  
Priority: Medium

Test Data:
- panel: 333 mm × 333 mm

Expected:
- area per panel = 0.110889 → aggregator returns 0.11 if single, or aggregated per sum then rounded.

---

## Gherkin / BDD Scenarios

Feature: Material surface area aggregation
  In order to calculate sheet purchase quantities
  As a user generating the BOM summary
  I want total area per material_id shown in m^2 rounded to two decimals

Background:
  Given the project uses "White Melamine" for carcass and "Oak Veneer" for doors

Scenario: BOM Summary lists separate totals per material
  Given the project contains White Melamine panels totalling 2.40 m^2
  And Oak Veneer panels totalling 1.00 m^2
  When I view the BOM Summary
  Then I see "White Melamine: 2.40 m^2"
  And I see "Oak Veneer: 1.00 m^2"

Scenario: Aggregation ignores material differences
  Given two panels with identical dimensions but different material_id
  When I view BOM Summary
  Then each material_id has its own total and values are not combined

Scenario: Totals rounded to two decimals
  Given panels produce a raw total area 0.221778 m^2
  When I view BOM Summary
  Then total is displayed as 0.22 m^2

Scenario: Mixed units are converted correctly
  Given one panel specified in mm (1000 × 600 mm) and another in m (1.2 × 0.8 m) using same material
  When I view BOM Summary
  Then total equals 1.56 m^2

Scenario: Deleting a panel updates totals
  Given initial White Melamine total is 1.20 m^2
  When I delete one panel contributing 0.60 m^2 and refresh BOM Summary
  Then total updates to 0.60 m^2

Scenario: Invalid panels are skipped and logged
  Given a panel is missing width value
  When aggregation runs
  Then that panel is not included and a warning is emitted

---

## Mapping to Acceptance Criteria
- "Area is calculated as Length × Width for every panel." — Covered by TC-0003-001, TC-0003-004, MAT-CTC-0001.
- "Items with different material_id are not summed together." — Covered by TC-0003-002 and BDD scenario "Aggregation ignores material differences."
- "The total is rounded to two decimal places." — Covered by TC-0003-003, TC-0003-007, MAT-CTC-0002.

---

## Edge Cases and Recommendations
- Decide and document canonical input units (prefer storing panel dims in mm internally and convert to m for area). Tests include unit conversion but implementation can enforce a single canonical unit to simplify.
- Decide rounding mode globally (bankers vs half-up). Tests assume round-half-up; adjust when spec dictates.
- For very small panels, consider thresholding to exclude negligible cut-offs under some configurable minimum area.
- Provide traceability metadata: for each aggregated material include list of panel ids and per-panel area for auditing and sheet planning.
- Consider adding "wastage factor" configuration (e.g., +10%) for purchasing; treat it as an optional post-aggregation step with its own tests.
- If sheets come in fixed sizes, consider an optimizer module (not covered here) that maps total m^2 to number/size of sheets — test separately.

---

## Test Data Summary
- Canonical units: mm input converted to m before area.
- Example panels:
  - 1000 × 600 mm -> 0.60 m^2
  - 1000 × 500 mm -> 0.50 m^2
  - 333 × 333 mm -> 0.110889 m^2 -> aggregated and rounded as needed

---

What I did: I produced a single Markdown test-case document for User Req 0003 that contains explicit assumptions, derived formulas, unit- and integration-level test cases (including component tests for a MaterialAreaAggregator), Gherkin scenarios, mapping to acceptance criteria, edge-case recommendations, and example test data.

What's next: I can (pick one) export these cases to CSV for your test-management tool, generate unit-test skeletons for your backend (pytest/JUnit/xUnit) that implement the MaterialAreaAggregator tests, or produce UI E2E scripts (Cypress/Playwright) to validate BOM Summary displays. Which output do you want me to generate now?