# Test Case Document — Component Req 0002: Component-Hardware Mapping

## Reference
User Req - 0002 - Hardware Association Logic  
(See: ../../user-requirements/0002 - Hardware Association Logic.md)

## Component Requirement
Given a set of Component objects (Doors, Drawers) that have been successfully placed in the model, when the applyHardwareRules() method scans these components, then the service must lookup the HardwareMappingConfiguration (a JSON or Map structure) to determine dependencies (e.g., Drawer → SlidePair + Handle) and inject these derivative items into the BOM stream.

Acceptance Criteria:
- The service must lookup the HardwareMappingConfiguration (a JSON or Map structure) to determine dependencies (e.g., Drawer → SlidePair + Handle) and inject these derivative items into the BOM stream.

---

## Purpose and Scope
This document defines unit and integration test cases for the backend service responsible for mapping placed Components to hardware items via a configurable HardwareMappingConfiguration and injecting those items into the BOM stream. Tests cover correct mapping, aggregation, config lookup behavior, error handling, and isolation from unrelated services.

---

## Assumptions (explicit)
- Components are domain objects with at least: id, type (e.g., "Door", "Drawer"), dimensions (height, width, depth), and metadata.
- HardwareMappingConfiguration is provided as an injected dependency (JSON file, in-memory Map, or config service). Example mapping:
  {
    "Door": [{"sku":"STD-HINGE-01","qtyRule":"by_height"},{"sku":"HANDLE-STD-25","qty":1}],
    "Drawer": [{"sku":"SLIDE-PAIR-450","qty":1,"unit":"pair"},{"sku":"HANDLE-STD-25","qty":1}]
  }
- qtyRule "by_height" indicates a rule engine or well-known helper function must be applied (e.g., hinge thresholds).
- applyHardwareRules(components, config) -> returns list of HardwareLine { sku, name, quantity, unit, sourceComponentIds } OR injects lines into an existing BOM stream object.
- Tests will treat applyHardwareRules as pure function when possible; external services (DB, SKU resolution) are mocked.
- Quantity aggregation behavior: service aggregates identical SKUs across components by default (unless configuration specifies per-instance lines).
- Error handling: missing mapping entries should be tolerated (skip or log) and not crash the service; mapping schema validation exists and returns explicit errors for malformed mapping.
- Unit of measure for quantities: integer counts; for pairs the unit or metadata indicates "pair".

Tolerance:
- Numeric rules (e.g., hinge counts derived from height) are parametric in tests; test harness must allow configurable thresholds.

---

## Test Types
- Unit tests: validate pure-mapping logic, aggregation, config lookup, and error behaviors.
- Component-integration tests: validate applyHardwareRules integration with the BOM stream object, ensuring injection semantics.
- Negative tests: malformed mapping, missing mapping entries, invalid component data.
- Contract tests: mapping schema validation and deterministic outputs.
- Optional: property-based tests (random components) to assert invariants (no negative quantities, aggregation correctness).

---

## Test Data Examples (quick reference)
- Door_small: {id:"door-1", type:"Door", height:900}
- Door_medium: {id:"door-2", type:"Door", height:1500}
- Drawer_1: {id:"drawer-1", type:"Drawer", width:400}
- HardwareMappingConfiguration sample (JSON):
  {
    "Door": [
      {"sku":"STD-HINGE-01", "rule": "hinges_by_height"},
      {"sku":"HANDLE-STD-25", "qty": 1}
    ],
    "Drawer": [
      {"sku":"SLIDE-PAIR-450", "qty": 1, "unit": "pair"},
      {"sku":"HANDLE-STD-25", "qty": 1}
    ]
  }

---

## Structured Test Cases

ID: CTC-0002-01  
Title: applyHardwareRules() — Basic mapping lookup and injection (happy path)  
Related Requirement: Component Req - 0002 / User Req - 0002  
Priority: High  
Test Type: Unit / Component

Preconditions:
- HardwareMappingConfiguration loaded in memory as shown in sample.
- applyHardwareRules accepts (components, config) and returns hardwareList or mutates BOM stream.

Test Data:
- components = [Door_small]
- config contains mapping for "Door" as sample.

Steps:
1. Call result = applyHardwareRules(components, config).
2. Inspect returned hardwareList or BOM stream.

Expected Result:
- result contains:
  - SKU "STD-HINGE-01" with quantity computed by hinge rule for height 900 -> 2.
  - SKU "HANDLE-STD-25" with quantity 1.
- Each hardware line includes sourceComponentIds = ["door-1"] (or equivalent linkage).
- No external DB calls occurred (mocked services not invoked).

Postconditions:
- None.

---

ID: CTC-0002-02  
Title: Aggregation of identical hardware across multiple components  
Related Requirement: Aggregation / deduplication behavior  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Config as sample; aggregation enabled (default behavior).

Test Data:
- components = [Door_small, Door_small_copy, Drawer_1]

Steps:
1. Call applyHardwareRules(components, config).
2. Inspect returned hardware list.

Expected Result:
- Hinges aggregated: two doors (2 hinges each) produce SKU "STD-HINGE-01" with quantity = 4.
- Handles aggregated: two doors + one drawer => total "HANDLE-STD-25" qty = 3.
- Slides: one entry "SLIDE-PAIR-450" with qty = 1 (unit = pair).
- Each hardware line includes a list of sourceComponentIds.

Postconditions:
- None.

---

ID: CTC-0002-03  
Title: Mapping configuration lookup only — no DB or external I/O performed during mapping  
Related Requirement: Pure config-driven mapping (acceptance criterion)  
Priority: High  
Test Type: Unit / Isolation

Preconditions:
- Inject mocks for any repository or external SKU service.

Steps:
1. Call applyHardwareRules(components, config) with mocks in place.
2. Assert mapping results are correct.
3. Assert mocks had zero interactions.

Expected Result:
- applyHardwareRules used only the provided config and input components.
- No repository/DB/external service calls occurred.

Postconditions:
- None.

---

ID: CTC-0002-04  
Title: Behavior when mapping entry missing for a component type (graceful handling)  
Related Requirement: Robustness / tolerance for missing mapping entries  
Priority: High  
Test Type: Negative / Unit

Preconditions:
- Config intentionally missing "CustomDoor" mapping.

Test Data:
- components = [{id:"cd-1", type:"CustomDoor", height:1000}]

Steps:
1. Call applyHardwareRules(components, config).
2. Capture result and logs/warnings.

Expected Result:
- No hardware lines added for "CustomDoor".
- Method returns successfully (no uncaught exception).
- A warning or validation item is logged/returned indicating missing mapping for type "CustomDoor".
- No DB calls.

Postconditions:
- None.

---

ID: CTC-0002-05  
Title: Config schema validation — malformed mapping detected and reported  
Related Requirement: Config validation / safe-fail behavior  
Priority: Medium  
Test Type: Unit / Contract

Preconditions:
- Provide malformed config (e.g., mapping entry missing sku or qty/rule fields).

Test Data:
- config = {"Door": [{"name":"Hinge without sku"}]}

Steps:
1. Start service or call configValidator(config) if available.
2. Attempt to call applyHardwareRules(components, config).

Expected Result:
- configValidator returns schema errors describing the problem.
- applyHardwareRules either throws a well-documented exception or returns an error result indicating invalid config.
- No partial hardware injection occurs.

Postconditions:
- Replace config with valid one.

---

ID: CTC-0002-06  
Title: Quantity rule evaluation — hinge-by-height rule invoked correctly for thresholds  
Related Requirement: Rule-driven quantity calculation  
Priority: High  
Test Type: Unit / Rule engine

Preconditions:
- The rule "hinges_by_height" is implemented and thresholds are configurable.

Test Data:
- Door_small (900 mm), Door_medium (1500 mm), Door_tall (2200 mm)

Steps:
1. Call applyHardwareRules for each door individually or combined.
2. Inspect hinge quantities per door.

Expected Result:
- 900 mm -> 2 hinges; 1500 mm -> 3 hinges; 2200 mm -> 4 hinges (or according to configured thresholds).
- Result lines reference which door contributed which quantity.

Postconditions:
- None.

---

ID: CTC-0002-07  
Title: Deleting a component updates/injects BOM removal events (integration with BOM stream)  
Related Requirement: Deletion propagation acceptance criterion from User Req 0002  
Priority: High  
Test Type: Integration / End-to-end

Preconditions:
- BOM stream or BOM object available to accept injected changes.

Test Data:
- Start with components = [Door_small, Drawer_1].

Steps:
1. Apply hardware rules and inject into BOM; verify hardware counts.
2. Remove Door_small from components and call applyHardwareRules again (or call removeHardwareForComponent).
3. Inspect BOM to confirm hinge & handle counts decreased accordingly.

Expected Result:
- BOM no longer includes hinge quantities contributed by removed door.
- If 0 instances remain for a SKU, the BOM line is removed.
- SourceComponentIds updated appropriately.

Postconditions:
- Re-add component if needed.

---

ID: CTC-0002-08  
Title: Concurrency — deterministic mapping under concurrent apply calls (race conditions)  
Related Requirement: Stability / concurrency behavior  
Priority: Medium  
Test Type: Integration / Concurrency

Preconditions:
- The mapping service is thread-safe or external synchronization is tested.

Test Data:
- Multiple concurrent calls to applyHardwareRules with overlapping component sets.

Steps:
1. Simulate concurrent invocations (e.g., 10 parallel requests) that add/remove components and call applyHardwareRules.
2. Inspect final aggregated BOM results and verify consistency.

Expected Result:
- Final hardware counts reflect a consistent end-state (e.g., last-write-wins or merge strategy as defined).
- No duplicated or lost hardware lines due to race conditions.
- Service handles concurrent access without exceptions or corrupted state.

Postconditions:
- None.

---

ID: CTC-0002-09  
Title: Unit tests for mapping function determinism and idempotency  
Related Requirement: Deterministic operation / idempotency if applicable  
Priority: Medium  
Test Type: Unit

Preconditions:
- applyHardwareRules should be deterministic for given inputs.

Test Data:
- components list and config as canonical.

Steps:
1. Call applyHardwareRules(components, config) multiple times with same inputs.
2. Compare returned hardware lists for equality.

Expected Result:
- Repeated calls produce identical hardware lists (same SKUs, quantities, ordering or canonical ordering).
- Function is idempotent when called with same inputs.

Postconditions:
- None.

---

ID: CTC-0002-10  
Title: Unit test for mapping with quantity units (pairs vs pieces) and BOM representation  
Related Requirement: Correct unit handling in BOM (pair vs piece)  
Priority: Medium  
Test Type: Unit / Contract

Preconditions:
- Mapping entries contain "unit" field for slides (e.g., unit: "pair").

Test Data:
- components = [Drawer_1]

Steps:
1. Call applyHardwareRules(components, config).
2. Inspect the hardware line for slides.

Expected Result:
- Slides line has sku "SLIDE-PAIR-450", quantity 1, unit = "pair".
- BOM consumer can render "1 pair" or "2 pieces" according to BOM UI conventions.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Component → Hardware Mapping Service
  In order to populate BOMs with required hardware
  As the backend mapping service
  I want applyHardwareRules() to consult a HardwareMappingConfiguration and inject hardware items into the BOM stream

Background:
  Given a HardwareMappingConfiguration is loaded and valid

Scenario: Door maps to hinges and handle (happy path)
  Given a Door component of height 900 mm exists with id "door-1"
  When applyHardwareRules([door-1], config) is invoked
  Then the BOM stream shall include "STD-HINGE-01" quantity 2 with sourceComponentIds ["door-1"]
  And the BOM stream shall include "HANDLE-STD-25" quantity 1 with sourceComponentIds ["door-1"]

Scenario: Drawer maps to slide pair and handle
  Given a Drawer component exists with id "drawer-1"
  When applyHardwareRules([drawer-1], config) is invoked
  Then the BOM stream shall include "SLIDE-PAIR-450" quantity 1 unit "pair"
  And the BOM stream shall include "HANDLE-STD-25" quantity 1

Scenario: Missing mapping entry is handled gracefully
  Given a component type "CustomDoor" has no mapping entry in config
  When applyHardwareRules([customdoor], config) is invoked
  Then no hardware lines for "CustomDoor" shall be added
  And a warning shall be emitted indicating missing mapping

Scenario: Aggregation of hardware across multiple components
  Given three Door components of height 900 mm exist
  When applyHardwareRules(allDoors, config) is invoked
  Then the BOM shall contain "STD-HINGE-01" quantity 6 and "HANDLE-STD-25" quantity 3

Scenario: Malformed mapping configuration prevents partial injection
  Given the HardwareMappingConfiguration is malformed
  When the service validates config
  Then config validation shall fail with descriptive errors and no hardware shall be injected

---

## Mapping to Acceptance Criteria
- "Service must lookup HardwareMappingConfiguration to determine dependencies and inject into BOM stream" — Covered by CTC-0002-01, CTC-0002-03, CTC-0002-05.
- Behavior for drawers and doors and deletion propagation is exercised indirectly by integration tests (CTC-0002-02, CTC-0002-07).

---

## Implementation / Test Automation Guidance
- Unit test frameworks: JUnit + Mockito (Java), pytest + unittest.mock (Python), xUnit + Moq (.NET).
- Structure tests to inject an in-memory config and to mock any external dependencies (SKU service, DB).
- For config validation, use JSON Schema and unit tests that load example config files (valid and invalid).
- When applyHardwareRules mutates a BOM stream, use an in-memory BOM object for assertions.
- Provide test helpers to build Component objects quickly (test builders).
- For concurrency tests, use threading utilities or async test harnesses to simulate parallel calls.
- Use snapshot or canonicalized ordering when asserting equality of hardware lists to avoid brittle ordering failures — compare by SKU and aggregated quantity instead.

Example pseudo-code (Python-like):
  def test_apply_hardware_rules_basic():
      config = load_test_config()
      components = [Component(id="door-1", type="Door", height=900)]
      result = mapping_service.applyHardwareRules(components, config)
      assert find(result, "STD-HINGE-01").quantity == 2
      assert find(result, "HANDLE-STD-25").quantity == 1
      assert repo_mock.method_calls == []

---

## Edge Cases & Recommendations
- Define exact schema for HardwareMappingConfiguration and test round-trips of the config.
- Make hinge/quantity rules pluggable so tests can swap threshold tables.
- Decide aggregation semantics (per-SKU vs per-instance lines) and provide tests for both modes if supported.
- Ensure mapping preserves sourceComponentIds for traceability and repair operations.
- Provide logging and telemetry for missing mappings and mapping application duration.
- Include property-based tests (fuzz random components and configs) to validate invariants: quantities non-negative, aggregated quantity equals sum of per-component contributions.
- If SKU lookup requires DB (e.g., to enrich human-readable name), separate mapping step (pure) from SKU-enrichment step (I/O) and test both layers separately. The pure mapping must not call DB.

---

## Test Data Summary (quick reference)
- Door_small: {id:"door-1", type:"Door", height:900} -> Hinges=2, Handle=1
- Drawer_1: {id:"drawer-1", type:"Drawer"} -> Slides(pair)=1, Handle=1
- Malformed config example: missing sku or qty fields -> validation error

---

I created a comprehensive component-level test-case document for Component Req - 0002 that includes unit and integration test cases, Gherkin scenarios, mapping to acceptance criteria, implementation guidance, and recommended edge cases. Next I can (and will, if you want) produce automated unit-test skeletons in your preferred language (e.g., pytest, JUnit) using the supplied config examples, or export the test cases as CSV for test management import. Which output do you want me to generate now?