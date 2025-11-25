# Test Case Document — User Req 0005: Total Cost Estimation

## Requirement
The system must calculate a total estimated cost for the project by multiplying the quantity/area of items in the BOM against a stored unit price in the database.

Scenario:
- Given the database contains a price of $20 per m^2 for wood and $5 per unit for hinges.
- When the BOM is rendered.
- Then a "Total Cost" column should display the line item cost, and a "Grand Total" footer should display the sum of all line items.

Acceptance criteria:
- Formula: (Area × Price_m2) + (Quantity × Price_unit).
- If a price is missing (null), the system warns the user rather than showing 0.00.
- Currency symbols are displayed consistent with user locale.

---

## Assumptions (explicit)
- BOM line items have either an area (in m^2) or a quantity (unit count) or both:
  - Material/area items: area_m2 (numeric)
  - Hardware/unit items: quantity (integer)
- Price lookup:
  - For area materials: price_per_m2 stored in prices table keyed by material_id or price_type.
  - For unit items: price_per_unit stored in prices table keyed by sku or hardware_id.
- Line cost calculation:
  - If both area and unit pricing apply to a line, lineCost = (area_m2 × price_per_m2) + (quantity × price_per_unit).
  - If only area present: lineCost = area_m2 × price_per_m2.
  - If only quantity present: lineCost = quantity × price_per_unit.
- Currency:
  - Prices stored in DB include currency code (e.g., "USD", "EUR").
  - UI displays currency symbol and formatting according to user locale settings (e.g., en-US shows "$", fr-FR shows "€" with locale format).
- Rounding:
  - Line costs and Grand Total are displayed with two decimal places (rounded half-up) but internal sums use higher precision until final display rounding.
- Missing price:
  - If price_per_m2 or price_per_unit is NULL for a line that needs it, the UI displays a visible warning icon/text for that line and excludes the line from the numeric Grand Total until the price is provided; or optionally the line is included in total only after user confirmation — tests will assert the system warns and does not silently treat price as zero.
- Taxes/discounts not in scope for base requirement (unless product extends).
- Pricing lookups should be performed server-side (backend) when rendering BOM, but mapping functions should be testable as pure functions.

---

## Derived examples / sample data
- DB:
  - material price: material_id = MAT-WOOD, price_per_m2 = 20.00 USD
  - hinge price: sku = STD-HINGE-01, price_per_unit = 5.00 USD
- BOM lines:
  - Line A: White Melamine panels totalArea = 1.56 m^2, material_id = MAT-WOOD -> lineCost = 1.56 × 20.00 = 31.20 USD
  - Line B: Hinges totalQty = 4, sku = STD-HINGE-01 -> lineCost = 4 × 5.00 = 20.00 USD
- Grand Total = 31.20 + 20.00 = 51.20 USD
- Display (en-US): $31.20, $20.00, Grand Total $51.20

---

## Test Types and Scope
- UI acceptance / E2E tests: Verify line costs and Grand Total display correctly on BOM render, currency formatting per locale, and warnings for missing prices.
- Backend/unit tests: CostCalculationService pure functions calculate line and total costs given BOM lines and price lookup map; handling of missing prices.
- Integration tests: Service uses DB price table via DAO; parameterized queries; null price handling; currency propagation.
- Negative/edge tests: missing price(s), mixed currencies across items, extremely large area/quantity (overflow), precision/rounding edge cases.
- Security tests: ensure price queries use parameterized statements.
- Performance sanity: ensure cost computation scales on large BOMs.

---

## Structured Test Cases

ID: TC-0005-001  
Title: Line item cost and grand total calculated correctly (happy path)  
Related Requirement: User Req - 0005  
Priority: High  
Test Type: Functional / Acceptance

Preconditions:
- Database contains:
  - MAT-WOOD.price_per_m2 = 20.00 USD
  - STD-HINGE-01.price_per_unit = 5.00 USD
- BOM contains:
  - White Melamine (material_id = MAT-WOOD): area_m2 = 1.56
  - Hinges (sku = STD-HINGE-01): quantity = 4

Steps:
1. Render BOM (UI) or call backend BOM endpoint.
2. Inspect line item "Total Cost" column and Grand Total footer.

Expected Result:
- White Melamine line shows Total Cost = 1.56 × 20.00 = 31.20 USD displayed as "$31.20" (locale en-US).
- Hinges line shows Total Cost = 4 × 5.00 = 20.00 USD displayed as "$20.00".
- Grand Total footer shows "$51.20".
- Numeric equality verified: 31.20 + 20.00 = 51.20.

Postconditions:
- None.

---

ID: TC-0005-002  
Title: Missing price triggers visible warning and not silently shown as $0.00  
Related Requirement: Warn user when price is missing  
Priority: High  
Test Type: Functional / Negative

Preconditions:
- Database: MAT-WOOD.price_per_m2 = NULL (missing)
- BOM contains White Melamine area_m2 = 1.56.

Steps:
1. Render BOM.
2. Inspect White Melamine line and Grand Total.

Expected Result:
- White Melamine line shows a warning indicator (icon and tooltip "Price missing for material MAT-WOOD").
- Line Total Cost cell shows blank or "—" or "Price missing", not "$0.00".
- Grand Total excludes the line (or UI displays Grand Total with note "Some items missing prices — total incomplete") — verify product-defined behavior (test asserts chosen behavior).
- Optionally a link/action to "Provide price" is available.

Postconditions:
- Restore price or remove test data.

---

ID: TC-0005-003  
Title: Currency formatting respects user locale (en-US and fr-FR examples)  
Related Requirement: Currency symbol/format per locale  
Priority: High  
Test Type: UI / Localization

Preconditions:
- Prices in DB in USD.
- User locale set to en-US, then to fr-FR.

Test Data:
- Reuse sample BOM (1.56 m^2 wood @20, 4 hinges @5).

Steps:
1. Set user locale to en-US and render BOM.
2. Assert currency formatting and symbol.
3. Set user locale to fr-FR and render BOM.

Expected Result:
- en-US: values shown as "$31.20", "$20.00", "Grand Total $51.20".
- fr-FR: values shown as "31,20 $US" or "31,20 $", or follow product-defined formatting; verify that formatting matches locale settings (currency symbol placement, decimal separator).
- No loss of numeric precision due to formatting.

Postconditions:
- Revert locale.

---

ID: TC-0005-004  
Title: Line cost calculation when both area and quantity apply to same line  
Related Requirement: Combined formula support  
Priority: Medium  
Test Type: Unit / Integration

Preconditions:
- BOM line includes both area_m2 and quantity (e.g., a panel bundled with hardware in same logical line).
- DB: material price_per_m2 = 20.00; unit price for an extra item present = 3.00.

Test Data:
- area_m2 = 2.5, quantity = 2, price_per_m2 = 20.00, price_per_unit = 3.00.

Steps:
1. Compute line cost via service or UI.

Expected Result:
- lineCost = (2.5 × 20.00) + (2 × 3.00) = 50.00 + 6.00 = 56.00
- Display shows "$56.00" and Grand Total includes this value.

Postconditions:
- None.

---

ID: TC-0005-005  
Title: Rounding behaviour — line sums and grand total rounding stability  
Related Requirement: Rounding to two decimals for display  
Priority: Medium  
Test Type: Numeric / Unit

Preconditions:
- Several lines produce values with fractions of cents after calculations.

Test Data:
- Line 1: area_m2 = 0.333, price_per_m2 = 20.00 -> raw = 6.66
- Line 2: area_m2 = 0.333, price_per_m2 = 20.00 -> raw = 6.66
- Expected raw sum = 13.32 -> display 13.32

Steps:
1. Render BOM and inspect numeric values.
2. Verify approach: internal high-precision sum then round final displayed values; or round per-line then sum (assert product design).

Expected Result:
- Displayed per-line values and Grand Total match chosen rounding policy:
  - If rounding per-line then summing: line displays 6.66, total 13.32.
  - If summing raw then rounding: same result here; tests should verify consistency and no off-by-one-cent anomalies.

Postconditions:
- None.

---

ID: TC-0005-006  
Title: Mixed currencies present in BOM — clear indication or conversion policy applied  
Related Requirement: Currency handling across items (edge)  
Priority: Medium  
Test Type: Integration / UI

Preconditions:
- DB contains:
  - MAT-WOOD priced in USD 20.00
  - SPECIAL_HANDLE priced in EUR 10.00
- User locale USD.

Steps:
1. Render BOM showing items priced in different currencies.
2. Inspect line costs and Grand Total behavior.

Expected Result:
- Either:
  - System blocks mixed-currency Grand Total and shows a warning requiring currency normalization, OR
  - System converts all prices to user's display currency using latest exchange rates (test requires mocking FX service), and Grand Total displayed in single currency.
- Tests must assert whichever policy product defines. If conversion occurs, verify rates applied and displayed rate/source tooltip.

Postconditions:
- None.

---

ID: TC-0005-007  
Title: Missing price warnings are logged and surfaced to admin (operational awareness)  
Related Requirement: Alerting/visibility for missing data  
Priority: Low / Optional  
Test Type: Integration / Operational

Preconditions:
- Several line items missing prices.

Steps:
1. Render BOM.
2. Verify that warning is visible per-line and that an operational log entry or event is produced (if system requires).

Expected Result:
- UI shows warning for each missing-price line.
- Backend logs an audit/telemetry event listing missing price item IDs for monitoring.

Postconditions:
- Clear test logs.

---

ID: TC-0005-008  
Title: Performance sanity: compute costs for large BOM (1000+ lines)  
Related Requirement: Performance / Scalability  
Priority: Low / Optional  
Test Type: Performance

Preconditions:
- Test DB with project BOM containing 1000+ line items.

Steps:
1. Trigger BOM render that invokes cost calculation.
2. Measure response time and memory.

Expected Result:
- Cost computation completes within acceptable threshold (define per system, e.g., < 1s server-side for 1000 lines).
- No errors or timeouts.

Postconditions:
- None.

---

ID: TC-0005-009  
Title: Security: price lookup uses parameterized queries to prevent SQL injection  
Related Requirement: Security / DB queries  
Priority: High  
Test Type: Security / Integration

Preconditions:
- Data access layer to prices exists.

Steps:
1. Inspect DAO/repository code or run injection attempt with malicious project/sku values.
2. Verify query uses bound parameters.

Expected Result:
- No SQL injection possible; parameterized statements used.
- Malicious input treated as parameter value, not SQL.

Postconditions:
- None.

---

## Component / Unit Test Cases (CostCalculationService)

ID: COST-CTC-0005-01  
Title: calculateLineCost(line, priceMap) pure function (unit)  
Priority: High  
Test Type: Unit

Preconditions:
- CostCalculationService exposes a pure function:
  calculateLineCost(line: BomLine, priceMap: PriceMap) -> { lineCost: Decimal, missingPrices: [fields] }

Test Data:
- line: { id: "L1", material_id: "MAT-WOOD", area_m2: 1.56, quantity: 0 }
- priceMap: { MAT-WOOD: { price_per_m2: 20.00 } }

Steps:
1. Call calculateLineCost.

Expected Result:
- Returns lineCost = 31.20
- missingPrices = []

Postconditions:
- None.

---

ID: COST-CTC-0005-02  
Title: calculateLineCost handles missing price and reports which price missing  
Priority: High  
Test Type: Unit

Preconditions:
- line requires price_per_m2 not present in priceMap.

Steps:
1. Call calculateLineCost with priceMap missing MAT-WOOD.

Expected Result:
- lineCost returned as null or 0 depending on API (test expects null and missingPrices includes "MAT-WOOD.price_per_m2").
- Consumer (UI/service) will render warning; unit tests assert missingPrices contents.

Postconditions:
- None.

---

ID: COST-CTC-0005-03  
Title: calculateGrandTotal(lines, priceMap) aggregates line costs and returns totals and missing price summary  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Several lines with some missing prices.

Steps:
1. Call calculateGrandTotal(lines, priceMap).

Expected Result:
- Returns:
  - lineCosts array with per-line numeric or null + missing price flags
  - grandTotal numeric sum over lines that have valid costs
  - missingPriceSummary listing affected line ids

Postconditions:
- None.

---

ID: COST-CTC-0005-04  
Title: Price lookup component queries DB and returns price map (integration)  
Priority: High  
Test Type: Integration

Preconditions:
- Test DB seeded with price rows.

Steps:
1. Call PriceRepository.getPricesForProject(projectId) or PriceService.getPriceMap(keys).
2. Inspect returned price map.

Expected Result:
- Correct key->price mapping returned for materials and SKUs used in BOM.
- Query parameterized and scoped.

Postconditions:
- Cleanup test data.

---

## Gherkin / BDD Scenarios

Feature: Total cost estimation for BOM items
  In order to provide an estimated project cost
  As a user viewing the BOM
  I want per-line costs and a Grand Total calculated from prices in the database

Background:
  Given prices: MAT-WOOD = 20.00 USD per m^2, STD-HINGE-01 = 5.00 USD per unit

Scenario: Display line costs and grand total
  Given BOM has White Melamine area 1.56 m^2 and Hinges quantity 4
  When I view the BOM
  Then the Wood line shows $31.20 and the Hinges line shows $20.00
  And the Grand Total should show $51.20

Scenario: Missing price warns user and excludes line from numeric total
  Given MAT-WOOD price is NULL
  When I view the BOM
  Then the Wood line displays a "Price missing" warning and not "$0.00"
  And the Grand Total indicates incomplete total due to missing prices

Scenario: Currency formatting follows user locale
  Given user locale is fr-FR
  When I view the BOM with USD prices
  Then amounts are displayed using fr-FR formatting and currency notation

Scenario: Mixed currencies handled per policy (conversion or blocking)
  Given BOM contains items priced in multiple currencies
  When I view the BOM
  Then the system either converts to a single display currency using exchange rates or shows a warning that Grand Total cannot be computed across currencies

---

## Mapping to Acceptance Criteria
- Formula (Area × Price_m2) + (Quantity × Price_unit) — covered by TC-0005-001, TC-0005-004, COST-CTC-0005-01.
- Missing price warning — covered by TC-0005-002, COST-CTC-0005-02.
- Currency symbol and locale formatting — covered by TC-0005-003 and relevant BDD scenarios.

---

## Edge Cases and Recommendations
- Decide a canonical policy for mixed currencies: either force conversion to user's display currency (requires FX service + tests) or require consistent currency for Grand Total and display per-currency subtotals with a warning. Tests included both branches.
- Decide whether missing-price lines are excluded from numeric Grand Total or cause the Grand Total to be flagged/incomplete; document and ensure tests assert chosen behavior.
- Persisted prices may have effective-dates; consider versioning/pricing date when computing project estimates (e.g., price at time of estimate vs current price) — add tests if required.
- Provide traceability: include price source and timestamp in UI (e.g., "Price (MAT-WOOD): $20.00 — updated 2025-11-01").
- Add audit logs when prices are updated to enable historical cost recomputation for saved projects.
- If taxes or shipping apply, these should be separate features with their own test sets.
- For very large sums, use decimal/numeric types with sufficient precision to avoid rounding/overflow errors.

---

## Implementation / Test Automation Guidance
- Backend:
  - Implement CostCalculationService with methods:
    - calculateLineCost(line, priceMap)
    - calculateGrandTotal(lines, priceMap)
  - PriceRepository returns a map keyed by material_id/sku with price and currency.
  - Use BigDecimal/Decimal types for monetary math.
  - Ensure parameterized queries for price lookups.
- UI:
  - Display per-line cost and Grand Total with explicit currency and two-decimal formatting per locale.
  - Show warnings for missing prices with actionable UI (link to price editor).
- Tests:
  - Unit tests for CostCalculationService using fixed price maps (pytest/Java JUnit/xUnit).
  - Integration tests with test DB seeded and transactional rollback.
  - E2E tests to assert UI rendering and locale formatting (Cypress/Playwright).
- Example pseudo-code (Python-like):
  def test_calculate_grand_total_excludes_missing_prices():
      lines = [ {... area 1.56, material: MAT-WOOD }, { quantity: 4, sku: STD-HINGE-01 } ]
      priceMap = { 'STD-HINGE-01': 5.00 }  # MAT-WOOD missing
      result = costService.calculateGrandTotal(lines, priceMap)
      assert result.lineCosts['MAT-WOOD'] is None
      assert result.grandTotal == 20.00
      assert 'MAT-WOOD' in result.missingPrices

---

## Test Data Summary (quick reference)
- Prices:
  - MAT-WOOD → 20.00 USD per m^2
  - STD-HINGE-01 → 5.00 USD per unit
- Example BOM:
  - White Melamine area = 1.56 m^2 → $31.20
  - Hinges quantity = 4 → $20.00
  - Grand Total = $51.20

---

What I did: I converted User Req - 0005 into a single Markdown test-case document with assumptions, derived numeric examples, structured test cases (UI, unit, integration, security, performance), component-level unit tests for CostCalculationService, Gherkin scenarios, mappings to acceptance criteria, edge-case recommendations, and automation guidance.

What's next: I can export this document as CSV for test-management import, generate unit-test skeletons for CostCalculationService in your preferred language (e.g., pytest, JUnit), or produce Cypress/Playwright E2E scripts to validate the BOM UI behavior — tell me which output and target framework you prefer and I will produce the code next.