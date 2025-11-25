# Test Case Document — Component Req 0005: Composite Pricing Calculator

## Reference
User Req - 0005 - Total Cost Estimation  
(See: ../../user-requirements/0005 - Total Cost Estimation.md)

## Component Requirement
Given a populated BOM object containing both Panel items (measured in area) and Hardware items (measured in units), when the calculateProjectCost() method is triggered, then the service must inject the PricingStrategy to fetch current unit costs from the DB, apply the math (Qty × UnitPrice) OR (Area × m^2Price) respectively, and reduce the stream to a single BigDecimal total.

Acceptance Criteria:
- The service must use PricingStrategy/PriceRepository to obtain prices for each BOM line (area price or unit price).
- For each BOM line the correct formula is applied:
  - For area-based items: lineCost = area_m2 × price_per_m2
  - For unit-based items: lineCost = quantity × price_per_unit
  - For combined items both terms apply and are added.
- All line costs are summed (with BigDecimal / high-precision) into a single BigDecimal project total.
- Missing prices cause the service to surface a missing-price indicator (exception, validation list, or warning) rather than silently treating price as zero (test to assert configured behavior).
- PricingStrategy is injected and mocked in unit tests; no direct DB queries by the calculator itself beyond the strategy.
- Currency/currency consistency: calculator returns numeric total in a single currency context (tests assert currency code propagation or error if mixed currencies).

---

## Assumptions (explicit)
- BOM object consists of BOMLine items with at least:
  - id: string
  - type: enum { PANEL, HARDWARE, COMPOSITE }
  - material_id or sku: string
  - area_m2: decimal (may be null)
  - quantity: integer (may be null)
  - currency: optional per-line currency code (fallback to project currency)
- PricingStrategy interface (injected) exposes:
  - getPriceForMaterial(materialId): Price { price: BigDecimal, currency: String } or null if missing
  - getPriceForSku(sku): Price { price: BigDecimal, currency: String } or null if missing
  - Bulk lookup variant: getPricesForKeys(keys[]): Map<key, Price>
- calculateProjectCost(bom, pricingStrategy, options) returns:
  - result: { total: BigDecimal (or null when incomplete per policy), currency: String, lineCosts: Map<lineId -> BigDecimal|null>, missingPrices: List<{lineId, requiredKey}> }
- All arithmetic uses BigDecimal (or language-equivalent decimal) with controlled scale and rounding (e.g., scale 4 internal, round half-up for intermediate, final result scaled to 2 decimals for presentation).
- Policy choices required and parameterized for tests:
  - Missing-price policy: "exclude" (exclude lines from numeric total and report missing), or "fail" (throw exception), or "treat-zero" (not acceptable by acceptance criteria but included for robustness tests).
  - Mixed-currency policy: "error", "convert" (requires FX service), or "per-currency subtotals". Tests will cover "error" and "convert (mocked FX)" variants.
- PricingStrategy is responsible for DB access; CompositePricingCalculator must not perform DB queries directly. Tests will assert no repository calls from the calculator.

---

## Test Types & Scope
- Unit tests: pure function behavior with mocked PricingStrategy and optional FX service.
- Integration tests: wiring between CompositePricingCalculator and real or test PricingStrategy implementation that reads from a test DB.
- Negative tests: missing prices, null area/quantity, negative quantities, mixed currencies, extremely large numeric values.
- Concurrency tests: multiple concurrent calculateProjectCost calls (thread-safety).
- Contract tests: returned DTO schema and types (BigDecimal, currency string, lists).

---

## Test Data (canonical)
- Pricing:
  - MAT-WOOD -> 20.00 USD per m^2
  - STD-HINGE-01 -> 5.00 USD per unit
- BOM lines:
  - L1 (panel): id="L1", type=PANEL, material_id=MAT-WOOD, area_m2=1.56, quantity=null
  - L2 (hardware): id="L2", type=HARDWARE, sku=STD-HINGE-01, quantity=4, area_m2=null
  - L3 (composite): id="L3", type=COMPOSITE, material_id=MAT-WOOD, area_m2=2.0, sku=EXTRA_PAD (price missing or provided depending on test), quantity=2

Derived expected:
- L1 cost = 1.56 × 20.00 = 31.20
- L2 cost = 4 × 5.00 = 20.00
- If L3 has price_per_m2=20 and price_per_unit=3 -> L3 = (2.0 × 20) + (2 × 3) = 46.00
- Total = sum of line costs (31.20 + 20.00 + 46.00 = 97.20)

---

## Structured Test Cases

ID: CPTC-0005-01  
Title: calculateProjectCost — happy path with area + unit lines (single currency)  
Related Requirement: Component Req - 0005 / User Req - 0005  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- PricingStrategy mocked to return:
  - MAT-WOOD -> { price: 20.00, currency: "USD" }
  - STD-HINGE-01 -> { price: 5.00, currency: "USD" }

Test Data:
- BOM: [L1 (area 1.56 MAT-WOOD), L2 (qty 4 STD-HINGE-01)]

Steps:
1. Inject mocked PricingStrategy into CompositePricingCalculator.
2. Call result = calculateProjectCost(bom).

Expected Result:
- result.lineCosts:
  - L1 = 31.20
  - L2 = 20.00
- result.total = 51.20 (BigDecimal, high precision)
- result.currency = "USD"
- result.missingPrices = [] (empty)
- Assert PricingStrategy methods were called exactly for MAT-WOOD and STD-HINGE-01.
- No direct DB/repository calls from calculator.

Postconditions:
- None.

---

ID: CPTC-0005-02  
Title: Missing price for a material is reported and excluded from total (exclude policy)  
Related Requirement: Missing price warning acceptance criterion  
Priority: High  
Test Type: Unit / Negative

Preconditions:
- PricingStrategy returns:
  - MAT-WOOD -> null (missing)
  - STD-HINGE-01 -> {5.00, "USD"}
- Missing-price policy configured to "exclude & report".

Test Data:
- BOM: [L1 (area 1.56 MAT-WOOD), L2 (qty 4 STD-HINGE-01)]

Steps:
1. Call calculateProjectCost(bom).

Expected Result:
- L1 lineCost = null (or flagged missing)
- L2 lineCost = 20.00
- result.total = 20.00 (excludes missing-priced lines)
- result.missingPrices = [{ lineId: "L1", key: "MAT-WOOD", requiredPrice: "price_per_m2" }]
- Calculator did not throw exception (per "exclude" policy).
- UI-level behavior (outside this component) will render missing-warning for L1.

Postconditions:
- None.

---

ID: CPTC-0005-03  
Title: Missing price triggers exception when policy is "fail"  
Related Requirement: Configurable missing-price policy (alternate)  
Priority: Medium  
Test Type: Unit / Negative

Preconditions:
- PricingStrategy returns null for required key.
- Calculator configured with missing-price policy "fail".

Steps:
1. Call calculateProjectCost(bom).

Expected Result:
- Calculator throws defined exception (e.g., MissingPriceException) containing details about missing keys.
- No partial total returned.

Postconditions:
- None.

---

ID: CPTC-0005-04  
Title: Mixed currencies cause error when no FX configured (policy "error")  
Related Requirement: Currency consistency acceptance criterion  
Priority: High  
Test Type: Integration / Negative

Preconditions:
- PricingStrategy returns:
  - MAT-WOOD -> {20.00, "USD"}
  - SPECIAL_HANDLE -> {10.00, "EUR"}
- Calculator policy = "error on mixed currencies".

Steps:
1. Call calculateProjectCost(bom containing both lines).

Expected Result:
- Calculator returns result indicating mixed currency error or throws MixedCurrencyException per product policy.
- No numerical total returned unless conversion applied.

Postconditions:
- None.

---

ID: CPTC-0005-05  
Title: Mixed currencies converted with FX service (policy "convert") — mock FX rates  
Related Requirement: Currency conversion path (optional)  
Priority: Medium  
Test Type: Unit / Integration

Preconditions:
- PricingStrategy returns prices in different currencies.
- FXService mocked to return conversion rates (e.g., 1 EUR = 1.1 USD).
- Calculator configured to convert all prices into project currency "USD".

Test Data:
- MAT-WOOD: 20.00 USD
- SPECIAL_HANDLE: 10.00 EUR -> 11.00 USD after conversion
- BOM contains both types.

Steps:
1. Inject mocked PricingStrategy and FXService.
2. Call calculateProjectCost(bom, projectCurrency="USD").

Expected Result:
- Each line priced in USD after conversion.
- result.total equals sum of converted line costs (assert BigDecimal correctness).
- result.currency = "USD"
- All external calls (PricingStrategy, FXService) asserted invoked.

Postconditions:
- None.

---

ID: CPTC-0005-06  
Title: Composite line with both area and quantity computed correctly  
Related Requirement: Formula combining area and unit price  
Priority: Medium  
Test Type: Unit

Preconditions:
- PricingStrategy returns:
  - MAT-WOOD -> 20.00 USD
  - EXTRA_PAD -> 3.00 USD per unit

Test Data:
- L3 composite: area_m2 = 2.0 (MAT-WOOD) and quantity = 2 (EXTRA_PAD)

Steps:
1. Call calculateProjectCost including L3.

Expected Result:
- L3 lineCost = (2.0 × 20.00) + (2 × 3.00) = 46.00
- Included in result.total as expected.

Postconditions:
- None.

---

ID: CPTC-0005-07  
Title: Precision and rounding — internal high precision; final value scaled to 2 decimals for UI  
Related Requirement: Use BigDecimal for accurate summation and rounding for display  
Priority: Medium  
Test Type: Unit / Numeric

Preconditions:
- Lines produce fractional intermediate values leading to rounding decisions.

Test Data:
- Line1: area 0.333333, price 20.00 -> raw = 6.66666...
- Line2: area 0.666667, price 20.00 -> raw = 13.33334...
- Raw total = 20.00000... should be exactly 20.00 after proper summation/rounding

Steps:
1. Call calculateProjectCost and capture internal total (if available) and returned final total.
2. Assert internal arithmetic used BigDecimal and final scaled/rounded total equals expected exact result.

Expected Result:
- result.total computed accurately and rounding to 2 decimals returns 20.00.
- No cumulative floating point error.

Postconditions:
- None.

---

ID: CPTC-0005-08  
Title: Concurrency: thread-safety under parallel requests (sanity)  
Related Requirement: Service stability under concurrent calculates  
Priority: Low / Optional  
Test Type: Concurrency / Integration

Preconditions:
- PricingStrategy thread-safe or mocked for concurrent usage.
- Simulate N parallel calls (e.g., 50 threads) to calculateProjectCost with same BOM.

Steps:
1. Run concurrent calls and gather results.

Expected Result:
- All calls succeed and return identical totals.
- No shared-state corruption or data races in calculator.
- Performance within acceptable thresholds.

Postconditions:
- None.

---

ID: CPTC-0005-09  
Title: No DB access from calculator — PricingStrategy must be the only access point (isolation)  
Related Requirement: Acceptance criterion: strategy injection and no direct DB calls  
Priority: High  
Test Type: Unit / Isolation

Preconditions:
- If calculator class depends on repository, inject mocks/spies for repository and PricingStrategy.

Steps:
1. Call calculateProjectCost with mocked PricingStrategy and repository spies.
2. Assert repository spies have zero calls; PricingStrategy was used.

Expected Result:
- PricingStrategy methods called; repository/DAO mocks not invoked by CompositePricingCalculator.
- Confirms calculator delegates DB access solely to PricingStrategy.

Postconditions:
- None.

---

ID: CPTC-0005-10  
Title: Empty BOM returns zero total and empty missingPrices list (or configured behavior)  
Related Requirement: Robustness / empty input handling  
Priority: Medium  
Test Type: Unit

Preconditions:
- BOM = []

Steps:
1. Call calculateProjectCost(emptyBOM).

Expected Result:
- result.total = 0.00 (BigDecimal) and result.missingPrices = [].
- result.currency may be null or project default; assert product-specified behavior.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Composite Pricing Calculator
  In order to estimate project cost accurately
  As the pricing calculation service
  I want to fetch current unit prices using the PricingStrategy and compute the project total as the sum of area- and unit-based line costs

Background:
  Given PricingStrategy is available and injected into the CompositePricingCalculator

Scenario: Calculate total when all prices available
  Given BOM has a wood panel area 1.56 m^2 and 4 hinges
  And pricing: MAT-WOOD = 20.00 USD/m^2, STD-HINGE-01 = 5.00 USD/unit
  When calculateProjectCost() is invoked
  Then the line costs shall be 31.20 and 20.00 respectively
  And Grand Total shall be 51.20 USD

Scenario: Missing material price excluded and reported
  Given MAT-WOOD price is NULL
  When calculateProjectCost() is invoked
  Then the wood line shall be flagged as missing price
  And Grand Total shall include only lines with prices and missingPrices list contains the missing entry

Scenario: Mixed currencies cause an error when conversion is not configured
  Given one price is in USD and another in EUR and no FX service configured
  When calculateProjectCost() is invoked
  Then the calculator shall return an error indicating mixed currencies

Scenario: Conversion applied with FX service
  Given FX service provides conversion rates
  When calculateProjectCost() is invoked with convert policy
  Then all prices shall be converted and a single Grand Total returned in project currency

Scenario: Composite line uses both area and unit pricing
  Given a BOM line has area 2.0 m^2 and quantity 2 for extra pads
  And prices exist for both material and pad sku
  When calculateProjectCost() is invoked
  Then the line cost shall equal (area × m2Price) + (qty × unitPrice)

---

## Implementation / Test Automation Guidance

- Languages/frameworks: use BigDecimal (Java), Decimal (Python/decimal), or equivalent accurate decimal type in your stack.
- Design:
  - CompositePricingCalculator should accept injected PricingStrategy and optional FXService.
  - PricingStrategy should provide both single and bulk lookup APIs to optimize DB calls.
  - Calculator algorithm:
    1. Collect unique keys (material_ids and skus) from BOM.
    2. Call PricingStrategy.getPricesForKeys(keys) once (bulk) to minimize DB roundtrips.
    3. Iterate BOM lines, compute per-line cost using BigDecimal:
       - area_m2 × price.price_per_m2 (if present)
       - quantity × price.price_per_unit (if present)
    4. Track missing prices and respect configured missing-price policy.
    5. If currencies differ and conversion policy is "convert", call FXService.bulkConvert(prices, targetCurrency).
    6. Sum using BigDecimal with a consistent MathContext/scale.
    7. Return DTO: { total: BigDecimal, currency: String, lineCosts: Map, missingPrices: List }.
- Test practices:
  - Unit tests: mock PricingStrategy and FXService; assert calls (keys, invocation counts) and no direct repository calls.
  - Integration tests: provide a test PricingStrategy implementation that reads from an in-memory DB or fixture table; test parameterized queries and mapping.
  - Use data-driven tests for edge conditions (missing price, negative quantities, fractional areas).
  - For concurrency tests, use thread pools and assert deterministic identical outputs.
  - For numeric tests use strict BigDecimal equality or compare using expected scale.
- Example pseudo-code (Java-like):
  PriceMap prices = pricingStrategy.getPricesForKeys(Set.of("MAT-WOOD","STD-HINGE-01"));
  CalculationResult res = calculator.calculateProjectCost(bom, prices, projectCurrency="USD");
  assert res.getTotal().equals(new BigDecimal("51.20"));
- Ensure API contract is explicit about whether calculator returns pre-rounded totals or raw precise totals (recommended: return precise BigDecimal; UI formats to 2 decimals).

---

## Edge Cases & Recommendations
- Decide and document missing-price policy (exclude, fail, treat-zero). Tests above cover both exclude & fail; choose and make tests authoritative.
- Decide currency policy: mixed-currency handling often requires business decision; prefer conversion with explicit FX rates or per-currency subtotals with warning.
- Bulk price lookup is essential for performance — tests should assert that calculator uses bulk lookup (single call) rather than N per-line calls.
- If prices have effective dates or tiered pricing, extend PricingStrategy to accept date/context and add tests for time-based pricing.
- Maintain auditability: include price source and timestamp in calculation result for traceability; add tests asserting presence of metadata.
- Persist calculation snapshots for reproducible estimates: tests should assert that re-running with same price snapshot yields identical totals.
- Avoid rounding early: sum raw BigDecimal values then round final total for display.
- For very large BOMs, test for numeric overflow and choose numeric types with sufficient precision/scale.

---

## DTO / Contract Example

CalculationResult {
  total: BigDecimal,           // precise total in project currency
  currency: String,            // e.g., "USD"
  lineCosts: Map<string, BigDecimal|null>, // per-line cost or null if missing price
  missingPrices: List<{ lineId: string, key: string }>,
  priceFetchInfo?: { fetchedAt: ISO8601, source: string }
}

---

I prepared a detailed, executable test-case document for Component Req - 0005 (Composite Pricing Calculator). It includes explicit assumptions, canonical test data, ten structured test cases (unit, integration, negative, concurrency), BDD scenarios, implementation guidance, and a recommended DTO contract. Next I can (pick one): generate unit-test skeletons in your preferred language (JUnit/PyTest/xUnit) that implement the unit cases using mocks, or produce a small reference implementation sketch of the CompositePricingCalculator and PricingStrategy interfaces. Which would you like me to produce now?