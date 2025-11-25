# Test Case Document — Component: Backend Domain Layer (GeometryEngine)

## Reference
User Req - 0001 - Automatic Panel Dimension Calculation  
(See: ../../user-requirements/0001 - Automatic Panel Dimension Calculation.md)

## Component Requirement
Given a generic FurnitureBox entity defined by vector dimensions (x, y, z) and a MaterialStrategy indicating construction method (e.g., "Sides-Surround-Top"), when the decomposeToParts() method is invoked, then the engine must return a list of PartDefinition objects where the geometric subtraction of material thickness is applied purely based on the math of the construction strategy (e.g., Top Width = Box Width − (2 × Thickness)), without accessing the database.

Acceptance Criteria:
- decomposeToParts() returns a list of PartDefinition objects computed only from FurnitureBox dimensions and the MaterialStrategy math (no DB access).

---

## Assumptions (explicit)
- The GeometryEngine runs in the backend domain layer and exposes a method:
  - decomposeToParts(box: FurnitureBox, strategy: MaterialStrategy) -> List<PartDefinition>
- FurnitureBox dimensions are provided as (height H, width W, depth D) in millimetres (mm).
- MaterialStrategy encodes the construction rules (e.g., "Sides-Surround-Top" means two side panels extend full H×D, top and bottom fit between sides so width subtracts 2×thickness).
- Thickness (t) is provided as a number in mm, may be integer or fractional (e.g., 18 or 12.5).
- PartDefinition contains at minimum: name/type (e.g., "Side Panel"), quantity, width (mm), height (mm), depth (mm), materialThickness (mm), and optionally join info or metadata.
- The engine is a pure computational component for these tests: it must not call repository/DB services. If the component normally depends on other services, they must be mocked; tests must assert those mocks were not called.
- Numeric comparisons should allow a small tolerance for floating-point arithmetic (suggest ±0.01 mm unless otherwise specified).

Derived sample values for the canonical box
- Box: H = 2000 mm, W = 1000 mm, D = 600 mm, thickness t = 18 mm
  - internal_width = W − 2×t = 964 mm
  - internal_height = H − 2×t = 1964 mm (if construction subtracts top+bottom thickness)
  - Side panels (2): H × D = 2000 × 600 mm (thickness attribute stored separately)
  - Top/Bottom (1 each): internal_width × D = 964 × 600 mm

---

## Test Types and Scope
- Unit tests (primary): verify pure math of decomposeToParts(), returned PartDefinition list content and values.
- Integration-like unit tests: ensure no database access or external I/O is performed (spy/mocks).
- Edge/negative tests: invalid dimensions or thickness, fractional thickness, zero/negative thickness, extremely large numbers.
- Strategy variants: different MaterialStrategy values produce expected formulas.

---

## Structured Test Cases (Unit/Component Level)

ID: CTC-0001  
Title: decomposeToParts() — Happy path with "Sides-Surround-Top" strategy  
Related Requirement: Component GeometryEngine requirement / User Req - 0001  
Priority: High  
Test Type: Unit / Functional

Preconditions:
- GeometryEngine instance available.
- No DB or external services invoked by this component (or they are replaced by mocks/spies).

Test Data:
- box = FurnitureBox(H=2000, W=1000, D=600)
- materialThickness = 18
- strategy = "Sides-Surround-Top"

Steps:
1. Call parts = GeometryEngine.decomposeToParts(box, strategy, thickness=18)
2. Inspect returned parts list.

Expected Result:
- parts is a list/array of PartDefinition objects.
- Contains exactly:
  - Side Panel — quantity: 2, height: 2000 mm, depth: 600 mm, thickness: 18 mm
  - Top Panel — quantity: 1, width: 964 mm, depth: 600 mm, thickness: 18 mm
  - Bottom Panel — quantity: 1, width: 964 mm, depth: 600 mm, thickness: 18 mm
- For each PartDefinition, numeric values are in mm and match derived values within tolerance.
- No repository/DB calls occurred (assert mock/spy not invoked).

Postconditions:
- None (pure computation).

---

ID: CTC-0002  
Title: decomposeToParts() recalculates correctly for changed box dimensions  
Related Requirement: Acceptance criterion for recalculation math  
Priority: High  
Test Type: Unit / Regression

Preconditions:
- Same environment as CTC-0001.

Test Data:
- box = FurnitureBox(H=2000, W=1200, D=600)
- thickness = 18
- strategy = "Sides-Surround-Top"

Steps:
1. Call parts = GeometryEngine.decomposeToParts(box, strategy, thickness=18)
2. Inspect Top/Bottom widths.

Expected Result:
- internal_width = 1200 − 2×18 = 1164 mm
- Top/Bottom PartDefinition width = 1164 mm
- Verify arithmetic invariant: internal_width + 2×thickness == outer_width (1164 + 36 = 1200)
- No DB calls.

Postconditions:
- None.

---

ID: CTC-0003  
Title: Output object schema — PartDefinition includes required fields and units  
Related Requirement: API contract / consumer expectations  
Priority: Medium  
Test Type: Unit / Contract

Preconditions:
- GeometryEngine available.

Test Data:
- Use canonical box / thickness.

Steps:
1. Call decomposeToParts(...)
2. For each returned PartDefinition, validate presence and types of fields:
   - name: string
   - quantity: integer
   - width: number (mm)
   - height: number (mm)
   - depth: number (mm)
   - materialThickness: number (mm)

Expected Result:
- All fields exist and are of expected type and unit (document mm).
- Dimensions values are non-negative numbers.

Postconditions:
- None.

---

ID: CTC-0004  
Title: No database or external service access during decomposition  
Related Requirement: Must not access DB (engine-only math)  
Priority: High  
Test Type: Unit / Isolation

Preconditions:
- If GeometryEngine has dependencies injected (e.g., repository), provide mocks/spies for them.

Steps:
1. Set up mocks for any repository/services and ensure they track invocations.
2. Call decomposeToParts(...) with canonical inputs.
3. Assert returned list correctness (as in CTC-0001).
4. Assert mocks have zero calls / zero interactions.

Expected Result:
- All external mocks show zero interactions.
- decomposeToParts is purely computational.

Postconditions:
- None.

---

ID: CTC-0005  
Title: Reject or error on invalid thickness producing non-positive internal dimension  
Related Requirement: Robustness / validation  
Priority: High  
Test Type: Negative / Unit

Preconditions:
- GeometryEngine should throw or return an error/validation structure for invalid geometry.

Test Data:
- box = FurnitureBox(H=2000, W=30, D=600)
- thickness = 20
- strategy = "Sides-Surround-Top"

Steps:
1. Call decomposeToParts(box, strategy, thickness=20)
2. Capture exception or return value.

Expected Result:
- The method returns a well-formed error/validation result or throws a defined exception (document exception type).
- No PartDefinition list with negative sizes is returned.
- No DB calls.

Postconditions:
- None.

---

ID: CTC-0006  
Title: Support fractional (non-integer) thickness and precision check  
Related Requirement: Numeric precision / units  
Priority: Medium  
Test Type: Unit / Numeric

Preconditions:
- GeometryEngine handles floating-point arithmetic.

Test Data:
- box = FurnitureBox(H=2000, W=1000, D=600)
- thickness = 12.5
- strategy = "Sides-Surround-Top"

Steps:
1. Call parts = decomposeToParts(box, strategy, thickness=12.5)
2. Inspect Top/Bottom widths.

Expected Result:
- internal_width = 1000 − 2×12.5 = 975.0 mm
- Top/Bottom width fields equal 975.0 mm (or 975 mm formatted by consumer).
- Sum check: internal_width + 2×thickness == outer_width within tolerance (975.0 + 25.0 == 1000.0).
- No DB calls.

Postconditions:
- None.

---

ID: CTC-0007  
Title: Different MaterialStrategy variants produce expected formulas (example: "Back-Inset", "Dado-Top")  
Related Requirement: Strategy-driven geometry formulas  
Priority: Medium  
Test Type: Unit / Strategy-combinatorial

Preconditions:
- GeometryEngine supports multiple strategies and has clear formula definitions per strategy.

Test Data and Steps (repeat per strategy):
- Example strategy A: "Sides-Surround-Top" -> Top width = W − 2×t
- Example strategy B: "Sides-Butt-Top" -> Top width = W − 2×t (same) OR may differ if join rules change; confirm spec.
- Example strategy C: "Back-Inset" -> Back panel may be inset inside full width/height by thickness on one or more sides (specify formula).

For each strategy:
1. Call decomposeToParts(box, strategy, thickness)
2. Assert each PartDefinition follows the formula defined by the strategy.

Expected Result:
- For every tested strategy, returned sizes match the strategy's mathematical rules.
- No DB calls.

Postconditions:
- None.

---

ID: CTC-0008  
Title: Large dimension inputs and performance sanity check (unit-level perf)  
Related Requirement: Stability / numeric limits  
Priority: Low / Optional  
Test Type: Unit / Performance

Preconditions:
- Use large but realistic inputs to reveal potential overflow or performance issues.

Test Data:
- box = FurnitureBox(H=1000000, W=1000000, D=1000000), thickness=18

Steps:
1. Call decomposeToParts(...) and measure execution time and correctness.

Expected Result:
- Method returns correct PartDefinition values (math scales).
- Execution time is reasonable for a pure computation (document a threshold for your environment).
- No DB calls.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios (Component-focused)

Feature: GeometryEngine.decomposeToParts() pure math decomposition
  In order to produce PartDefinition objects for BOM calculations
  As the backend domain GeometryEngine
  I want decomposeToParts() to compute part sizes from FurnitureBox dimensions and MaterialStrategy without external I/O

Background:
  Given a FurnitureBox H=2000 mm, W=1000 mm, D=600 mm and material thickness=18 mm

Scenario: Decompose returns expected parts for "Sides-Surround-Top"
  Given a MaterialStrategy "Sides-Surround-Top"
  When decomposeToParts(box, strategy, thickness=18) is invoked
  Then the result shall include:
    | part         | qty |
    | Side Panel   | 2   |
    | Top Panel    | 1   |
    | Bottom Panel | 1   |
  And Top Panel width shall equal 964 mm
  And No repository or database call shall be made during decomposition

Scenario: Decompose recalculates for changed box dimension
  Given a FurnitureBox with W=1200 mm
  When decomposeToParts(box, "Sides-Surround-Top", thickness=18) is invoked
  Then Top Panel width shall equal 1164 mm
  And internal_width + 2 * thickness = outer_width

Scenario: Decompose rejects invalid thickness
  Given a FurnitureBox with W=30 mm
  And thickness = 20 mm
  When decomposeToParts(...) is invoked
  Then the engine shall return an error indicating thickness too large to produce valid parts
  And no PartDefinition with negative dimension shall be returned

Scenario: Decompose supports fractional thickness
  Given thickness = 12.5 mm
  When decomposeToParts(...) is invoked
  Then Top Panel width shall equal 975.0 mm within tolerance

---

## Implementation / Unit Test Guidance (examples)

- Testing frameworks: use unit test frameworks appropriate to your backend (JUnit/Mockito for Java, pytest for Python, NUnit/xUnit for .NET).
- Use dependency injection to inject mocks/spies for any repository or service the GeometryEngine might accept — assert that those mocks are not called.
- Example pseudo-code (Python-like):

  def test_decompose_sides_surround_top_no_db_calls(mocker):
      # Arrange
      box = FurnitureBox(H=2000, W=1000, D=600)
      strategy = MaterialStrategy("Sides-Surround-Top")
      thickness = 18
      repo_mock = mocker.Mock()
      engine = GeometryEngine(repo=repo_mock)  # if engine accepts repo but should not use it
      # Act
      parts = engine.decomposeToParts(box, strategy, thickness)
      # Assert
      assert find_part(parts, "Top Panel").width == 964
      assert repo_mock.method_calls == []  # no DB calls

- For languages with strong typing, include schema/unit tests asserting PartDefinition types and serialized JSON representations (if used by consumers).

---

## Mapping to Acceptance Criteria
- "Geometric subtraction applied purely based on the math of the construction strategy" — covered by CTC-0001, CTC-0002, CTC-0006, and strategy tests (CTC-0007).
- "Without accessing the database" — covered by CTC-0004 and every test that asserts zero interactions with repository mocks.

---

## Edge Cases and Additional Recommendations
- Ensure negative and zero thickness are handled explicitly (reject or treat zero thickness as valid).
- Define and test behavior for extremely thin material (e.g., 0.1 mm) and extremely large thickness values (exceeding half of a box dimension).
- Test for NaN/Infinity inputs and non-numeric values (validate input types).
- Document exact error/exception types and messages for invalid input so tests can assert them deterministically.
- If strategies can reference additional parameters (e.g., inset offsets, rabbet depths), extend strategy tests to cover those parameters.
- Add property-based tests (fuzz) that generate random boxes and thickness values to verify invariant:
  internal_dimension + sum_of_relevant_thicknesses == outer_dimension (within tolerance).
- Add serialization/deserialization tests if PartDefinition is persisted or passed to other layers — ensure units (mm) are preserved.

---

## Test Data Summary (quick reference)
- Canonical case:
  - box: H=2000, W=1000, D=600, thickness=18
  - Derived: internal_width=964, internal_height=1964 (if applicable)
- Variation:
  - box W=1200 -> internal_width=1164
  - fractional thickness: 12.5 -> internal_width=975.0
  - invalid: outer_width=30, thickness=20 -> invalid

---

## What I have provided
- A complete component-level test-case document for the GeometryEngine decomposeToParts() functionality: unit test cases, BDD scenarios, test guidance and edge-case recommendations, and explicit checks that no database access occurs.
