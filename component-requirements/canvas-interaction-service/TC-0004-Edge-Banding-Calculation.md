# Test Case Document — User Req 0004: Edge Banding Calculation

## Requirement
The system must calculate the linear meters of edge banding required, distinguishing between visible edges (requiring premium banding) and non-visible edges (requiring standard or no banding).

Scenario:
- Given a shelf panel where only the front-facing edge is flagged as "Visible".
- When the BOM calculates the "Edge Banding" category.
- Then the length of the front edge is added to the "Premium Edging" total, and the other three sides are added to "Standard Edging" or ignored based on configuration.

Acceptance criteria:
- Linear meters are summed correctly per material type.
- The UI allows the user to toggle which edges of a rect are "taped/banded" in the design view, reflecting immediately in the BOM.

---

## Assumptions (explicit)
- Panels are rectangular with edges: front, back, left, right. Each edge has a length equal to its corresponding side.
- Dimensions are provided in millimetres (mm) internally; edge lengths for BOM are reported in linear metres (m). Conversion: metres = millimetres / 1000.
- Each edge has a visibility flag per panel: Visible (premium), NonVisible (standard), or NotBanded (no banding). Tests assume an enum or boolean flags accessible per edge.
- BOM Edge Banding categories:
  - Premium Edging (for visible edges)
  - Standard Edging (for non-visible edges)
  - Optionally "None" or omitted for NotBanded edges
- Configuration options:
  - Default behavior: Visible → Premium, NonVisible → Standard
  - Configurable: Option to ignore non-visible edges (i.e., only include premium)
  - Configurable: Treat "both-sides" banding where an edge flagged for both faces may double the length or be treated as single depending on spec (tests cover both behaviors if supported)
- Edge sharing: When two panels share the same edge (e.g., shelf rests against cabinet side), the system must define whether shared edges are banded once (no duplication) or per panel; tests include both expected behaviors depending on product rules (default: band edges per exposed edge only — shared internal seams are not banded).
- Linear totals must be rounded and displayed to two decimal places in metres (e.g., 1.25 m).
- UI toggle must cause immediate update in BOM (either instant or on explicit refresh if product requires; tests will support both modes but expect visible effect within user workflow).

---

## Derived formulas / examples
- Rectangular shelf panel: Width = 1000 mm (front/back), Depth = 300 mm (left/right).
- Edge lengths:
  - Front edge = 1000 mm = 1.0 m
  - Back edge = 1000 mm = 1.0 m
  - Left edge = 300 mm = 0.3 m
  - Right edge = 300 mm = 0.3 m
- Example: Only front edge Visible → Premium total = 1.00 m; Standard total = 1.60 m if standard includes the other three sides, or = 0.60 m if only left+right are standard and back ignored per config (tests parametric).

---

## Test Types and Scope
- UI E2E tests: toggling edge flags in design view, immediate BOM update, correct display and rounding.
- Unit/component tests: EdgeBandingCalculator or service functions produce correct banding lines from panel geometry and flags (pure math).
- Integration tests: multiple panels aggregation, shared edge handling, config variations.
- Negative/edge tests: zero-length edges, invalid flags, overlapping/shared edges, panels with rotated orientation, mixed units.
- Performance sanity: many panels aggregated.

---

## Structured Test Cases

ID: TC-0004-001  
Title: Single shelf front-edge visible — premium added, others standard (default config)  
Related Requirement: User Req - 0004  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Panel: Width = 1000 mm, Depth = 300 mm.
- Edge flags: front = Visible, back = NonVisible, left = NonVisible, right = NonVisible.
- Configuration: Default (include NonVisible as Standard).

Steps:
1. Open project with the panel.
2. Generate/refresh BOM and view Edge Banding category.

Expected Result:
- Premium Edging total = front length = 1000 mm → 1.00 m.
- Standard Edging total = back + left + right = 1000 + 300 + 300 = 1600 mm → 1.60 m.
- Values displayed in metres with two decimals:
  - Premium: 1.00 m
  - Standard: 1.60 m

Postconditions:
- None.

---

ID: TC-0004-002  
Title: Single shelf front-edge visible — non-visible ignored when config set to ignore non-visible edges  
Related Requirement: Configurable behavior to ignore non-visible edges  
Priority: High  
Test Type: Functional / Config

Preconditions:
- Same panel and flags as TC-0004-001.
- Configuration: "Ignore NonVisible Edges" = ON.

Steps:
1. Set config to ignore non-visible edges.
2. Refresh BOM.

Expected Result:
- Premium Edging total = 1.00 m.
- Standard Edging total = 0.00 m (or Standard row omitted).
- UI indicates non-visible edges are excluded (optional notice).

Postconditions:
- Revert config.

---

ID: TC-0004-003  
Title: Multiple panels aggregation per material type and banding category  
Related Requirement: Linear meters summed correctly per material type and category  
Priority: High  
Test Type: Integration / Acceptance

Preconditions:
- Panel A: Shelf front visible (1000 x 300 mm) — premium front 1.00 m
- Panel B: Shelf front visible (800 x 300 mm) — premium front 0.80 m
- Panel C: Back panel non-visible (1200 x 400 mm) — standard all edges (if flagged)
- Material grouping: panels use same edging material for premium and same for standard where applicable or config maps to material types (tests will parametrize different materials).

Steps:
1. Add panels to project and ensure their edge flags.
2. Generate BOM, inspect Edge Banding category.

Expected Result:
- Premium Edging total equals sum of premium edge lengths: 1.00 + 0.80 = 1.80 m (rounded 1.80).
- Standard Edging total sums all standard edges across panels (compute per panel, convert to metres, sum, round).
- Each material type (e.g., Premium PVC vs Standard PVC) shows totals separately if materials differ.

Postconditions:
- None.

---

ID: TC-0004-004  
Title: UI toggle updates BOM immediately (instant or refresh) — visible effect test  
Related Requirement: UI toggling reflection in BOM  
Priority: High  
Test Type: E2E / UI

Preconditions:
- A panel exists with front edge initially NonVisible.
- BOM initially shows Premium = 0.00 m.

Steps:
1. In design view toggle the front edge flag to Visible.
2. Observe BOM auto-update (or trigger refresh if UI requires).
3. Inspect Edge Banding totals.

Expected Result:
- Premium Edging increases by front length (1.00 m).
- Change reflected within a single user interaction (acceptable delay documented, e.g., < 2s).
- If auto-refresh disabled, the user-initiated refresh produces the expected update.

Postconditions:
- Revert toggle.

---

ID: TC-0004-005  
Title: Shared edge between two panels counted once when policy is "exposed-only"  
Related Requirement: Avoid double counting for shared internal edges (default policy)  
Priority: High  
Test Type: Integration / Edge-case

Preconditions:
- Two panels share a common edge (e.g., shelf touches side panel) and the shared edge is internal (not exposed).
- Both panels have the shared edge flagged as NonVisible or NotBanded.

Steps:
1. Calculate BOM Edge Banding totals.

Expected Result (default/exposed-only policy):
- Shared edge not counted (since not exposed); total excludes shared edge.
- If policy were "per-panel", test would expect double-count; document configured policy.

Postconditions:
- None.

---

ID: TC-0004-006  
Title: Per-panel banding policy (alternate configuration) — shared edges may be counted per panel  
Related Requirement: Config variation for banding policy  
Priority: Medium  
Test Type: Config / Integration

Preconditions:
- Same panels as TC-0004-005.
- Configuration: "Band per panel" = ON.

Steps:
1. Enable "Band per panel".
2. Generate BOM.

Expected Result:
- Shared edge counted twice (once per panel) and reflected in totals.
- Behavior aligns with configured policy.

Postconditions:
- Reset config.

---

ID: TC-0004-007  
Title: Zero-length or degenerate edge ignored and logged (defensive)  
Related Requirement: Robustness / validation  
Priority: Medium  
Test Type: Negative

Preconditions:
- Panel with width = 0 mm or depth = 0 mm.

Steps:
1. Include degenerate panel in project.
2. Generate BOM.

Expected Result:
- Edge with zero length contributes 0.00 m.
- Aggregator logs/records a warning for degenerate panel (panel id and reason).
- No exceptions thrown.

Postconditions:
- Remove degenerate panel.

---

ID: TC-0004-008  
Title: Rounding to two decimals for small accumulations and many small edges  
Related Requirement: Display rounding and numeric stability  
Priority: Medium  
Test Type: Numeric / Unit

Preconditions:
- 100 panels each with front edge length 333 mm and flagged Visible.

Steps:
1. Aggregate premium edge lengths: 100 × 333 mm = 33,300 mm → 33.3 m.
2. Ensure final display is rounded to two decimals: 33.30 m.

Expected Result:
- Accurate accumulation and rounding: 33.30 m displayed.
- No excessive floating-point drift.

Postconditions:
- None.

---

ID: TC-0004-009  
Title: Edge material mapping — premium and standard map to different SKU/material types in BOM  
Related Requirement: Per-material totals per banding type  
Priority: Medium  
Test Type: Functional / Integration

Preconditions:
- Premium edging uses material_id = MAT-PREMIUM-PVC, Standard edging uses MAT-STANDARD-PVC.
- Panels flagged accordingly.

Steps:
1. Generate BOM.
2. Inspect Edge Banding category grouped by material_id.

Expected Result:
- BOM displays separate lines per edging material:
  - MAT-PREMIUM-PVC: totalMeters (sum of premium edges in m)
  - MAT-STANDARD-PVC: totalMeters (sum of standard edges in m)
- Quantities shown in metres and appropriate units (e.g., "m" or "linear m").

Postconditions:
- None.

---

ID: TC-0004-010  
Title: Orientation/rotation correctness — edge lengths computed independent of panel rotation  
Related Requirement: Geometry correctness for rotated panels  
Priority: Low/Optional  
Test Type: Unit / Integration

Preconditions:
- Panel rotated (e.g., width and depth swapped visually, but stored dims unchanged).

Steps:
1. Create panel and rotate in design view.
2. Toggle edges and compute BOM.

Expected Result:
- Edge lengths computed from stored width/depth values, not UI rotation artifacts.
- Premium/Standard totals unaffected by rotation.

Postconditions:
- None.

---

## Component / Unit Test Cases (EdgeBandingCalculator)

ID: EBC-CTC-0004-01  
Title: EdgeBandingCalculator.calculate(panel, config) — happy path single panel  
Priority: High  
Test Type: Unit

Preconditions:
- EdgeBandingCalculator is a pure function: calculate(panel) -> List<BandingLine> or Map{material_id -> meters}.
- Panel object:
  - id: "shelf-1"
  - width_mm: 1000
  - depth_mm: 300
  - edges: {front: Visible, back: NonVisible, left: NonVisible, right: NonVisible}
- Config: default (include non-visible as standard, mapping Visible -> premium material_id).

Steps:
1. Call result = EdgeBandingCalculator.calculate(panel, config).

Expected Result:
- result contains:
  - BandingLine{material_id: MAT-PREMIUM, category: "Premium", length_m: 1.00, sourceEdges: ["front"], sourcePanelIds: ["shelf-1"]}
  - BandingLine{material_id: MAT-STANDARD, category: "Standard", length_m: 1.60, sourceEdges: ["back","left","right"], sourcePanelIds: ["shelf-1"]}
- No DB calls in calculation (assert no repository interaction if edge-band service injects mocks).

Postconditions:
- None.

---

ID: EBC-CTC-0004-02  
Title: EdgeBandingCalculator respects "Ignore NonVisible Edges" config  
Priority: High  
Test Type: Unit

Preconditions:
- Same panel as EBC-CTC-0004-01.
- Config: ignoreNonVisible = true.

Steps:
1. Call calculate(panel, config).

Expected Result:
- Only premium BandingLine returned with 1.00 m.
- No standard BandingLine present.

Postconditions:
- None.

---

ID: EBC-CTC-0004-03  
Title: Shared edge deduplication logic unit test (exposed-only policy)  
Priority: High  
Test Type: Unit

Preconditions:
- Two panel objects sharing the same geometric edge with flags indicating exposure only from one side (or internal no exposure).

Steps:
1. Call calculate aggregated over both panels with policy "exposed-only".
2. Inspect aggregated BandingLines.

Expected Result:
- Shared internal edge not double-counted; contribution only from exposed sides.
- SourcePanelIds indicate which panel(s) contributed.

Postconditions:
- None.

---

ID: EBC-CTC-0004-04  
Title: Numeric precision: mm -> m conversion and rounding to two decimals in aggregator (unit)  
Priority: Medium  
Test Type: Unit / Numeric

Preconditions:
- Panel with edge lengths summing to 1234 mm.

Steps:
1. Convert to metres: 1234 / 1000 = 1.234 m.
2. Round to two decimals for display: 1.23 m (round half-up or configured mode).

Expected Result:
- Calculation returns 1.234 internally; rounding function returns 1.23 for presentation.

Postconditions:
- None.

---

ID: EBC-CTC-0004-05  
Title: Invalid edge flag values produce validation error or are normalized (defensive)  
Priority: Medium  
Test Type: Negative / Unit

Preconditions:
- Panel with edge flag = null or invalid string.

Steps:
1. Call calculate(panel, config).

Expected Result:
- Calculator either:
  - Normalizes invalid flags to NotBanded or NonVisible and proceeds, or
  - Returns a validation error with a clear message.
- Behavior must be deterministic and documented.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Edge Banding Calculation
  In order to purchase correct lengths of edging material
  As a user designing furniture
  I want the BOM to show linear meters for premium and standard edging and to toggle edges in the design view

Background:
  Given a shelf panel Width=1000 mm Depth=300 mm

Scenario: Only front edge is visible and counted as premium
  Given the front edge is flagged Visible and other edges are NonVisible
  When I view the BOM Edge Banding category
  Then Premium Edging shall equal 1.00 m
  And Standard Edging shall equal 1.60 m

Scenario: Ignoring non-visible edges via configuration
  Given "Ignore NonVisible Edges" configuration is ON
  And only front edge is Visible
  When I view the BOM
  Then Premium Edging shall equal 1.00 m
  And Standard Edging shall be 0.00 m or omitted

Scenario: Toggling an edge updates BOM immediately
  Given the front edge is initially NonVisible
  When I toggle the front edge to Visible in the design view
  Then the BOM shall show Premium Edging increased by the front edge length

Scenario: Shared internal edge is not double-counted (exposed-only policy)
  Given two panels share an internal edge and neither side is exposed
  When the BOM computes edge banding totals
  Then the shared edge shall not contribute to premium or standard totals

Scenario: Edge banding totals grouped by edging material
  Given premium edges map to MAT-PREMIUM-PVC and standard to MAT-STANDARD-PVC
  When the BOM is shown
  Then totals shall be displayed grouped by material with linear metres for each

---

## Mapping to Acceptance Criteria
- "Linear meters are summed correctly per material type." — Covered by TC-0004-003, TC-0004-009, EBC-CTC-0004-01.
- "The UI allows the user to toggle which edges ... reflecting immediately in the BOM." — Covered by TC-0004-004 and BDD scenario "Toggling an edge updates BOM immediately."

---

## Edge Cases and Additional Recommendations
- Clarify policy for shared edges (default: exposed-only) and add tests for any alternate policies.
- Define whether an edge flagged Visible on one panel but adjacent to another panel should be considered exposed (e.g., reveal/clearance). Add tests for tolerance/clearance thresholds.
- Define how to treat mitred or angled panels (non-rectangular geometry): extend calculator for polylines and variable edge lengths.
- Consider an option "double-sided banding" where both faces of an edge require banding (count length ×2) — add tests if supported.
- Provide traceability: for each banding line, include sourcePanelIds and sourceEdgeNames to allow auditing and manual adjustments.
- Add an optional "wastage factor" percentage applied post-aggregation for purchasing; create separate tests for that behavior.
- Accessibility: ensure UI toggles are keyboard-navigable and changes announced to screen readers.
- Performance: test aggregation on large projects (thousands of panels) to ensure acceptable latency.

---

## Implementation / Test Automation Guidance
- UI automation: use Playwright or Cypress to simulate toggling edge flags in design view, then assert BOM table values (prefer selectors that identify edge toggles and BOM lines by data attributes).
- Backend/component tests: implement EdgeBandingCalculator unit tests (pytest / JUnit / xUnit), making the calculator a pure function that accepts panels and config and returns aggregated BandingLines. Mock any repository or DB enrichment and assert no calls during calculation.
- Data model: BandingLine DTO should include: material_id, material_name (optional), category ("Premium"/"Standard"), length_m (numeric), sourcePanelIds (list), sourceEdges (list), unit ("m").
- Rounding: keep internal values unrounded; apply rounding at presentation layer. Tests should assert both raw and rounded values depending on where rounding is performed.
- Use test builders to quickly create panels with dimensions and edge flags. Include fixtures for common configurations (exposed-only vs per-panel).
- For shared-edge detection, use panel adjacency/topology info; tests should include cases where adjacency data is missing (fallback to per-panel).

---

## Test Data Summary (quick reference)
- Panel canonical: width_mm=1000, depth_mm=300
  - Edge lengths: front=1000 mm (1.00 m), back=1000 mm (1.00 m), left=300 mm (0.30 m), right=300 mm (0.30 m)
- Example results (default include non-visible as standard):
  - Premium total (front only) = 1.00 m
  - Standard total (others) = 1.60 m

---

What I did: I converted User Req - 0004 into a single Markdown test-case document that includes explicit assumptions, numeric examples, 10 structured test cases (UI, integration, unit, negative), component-level unit tests for an EdgeBandingCalculator, Gherkin scenarios, mappings to acceptance criteria, edge-case recommendations, and automation guidance.

What's next: I can (pick one) export this document as a CSV for your test-management tool, generate unit-test skeletons for EdgeBandingCalculator in your preferred language (pytest/JUnit/xUnit), or produce Playwright/Cypress E2E scripts that toggle edge flags and assert BOM updates — tell me which output you want and which target language/framework or test tool to use.