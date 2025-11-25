# Test Case Document — Component Req 0003: SQL Aggregation View

## Reference
User Req - 0003 - Material Surface Area Aggregation  
(See: ../../user-requirements/0003 - Material Surface Area Aggregation.md)

## Component Requirement
Given the bom_line_items table is populated with individual cut pieces for a project, when the application requests the material summary, then a native SQL query or Projection must run a GROUP BY material_id operation to sum the area_mm2 and divide by 1,000,000, returning a streamlined Data Transfer Object (DTO) containing only MaterialName and TotalMetersSquared.

Acceptance Criteria:
- A SQL GROUP BY by material_id is executed to compute SUM(area_mm2) / 1,000,000.
- The returned DTO contains exactly: MaterialName (string) and TotalMetersSquared (numeric) for the requested project.
- The aggregation is scoped to the project (e.g., WHERE project_id = :projectId) so results are project-specific.

---

## Assumptions (explicit)
- Database table: bom_line_items columns include at least:
  - id (PK), project_id, material_id, length_mm, width_mm, area_mm2, quantity, created_at, ...
- A materials table exists (materials) with: material_id (PK), material_name, material_code, etc. MaterialName is resolved via join or the BOM item already carries material_name.
- area_mm2 is stored as integer (mm²) or numeric. If missing, area_mm2 should be computed as (length_mm × width_mm) per line prior to aggregation — tests include both precomputed and computed paths.
- Returned DTO must have exactly two properties:
  - material_name (string)
  - total_meters_squared (numeric), expressed in square metres (m^2) — i.e., SUM(area_mm2)/1_000_000.0
- Rounding: higher-level user requirement requires totals rounded to two decimal places in the UI. The SQL/projection may return unrounded numeric; tests will verify both unrounded DB result and final DTO rounding behavior where applicable. Tests will check both variants (raw DB value and presentation-rounded value) as specified by product policy.
- SQL dialect: tests should be adaptable for the target DB. Sample SQL provided uses ANSI SQL compatible expressions and also a Postgres-specific example for rounding.
- Security: Query should be parameterized to avoid SQL injection (use :projectId or prepared statements); tests should assert parameterization where possible.

---

## Example SQL (canonical)
Basic aggregation assuming area_mm2 exists:
SELECT m.material_name AS material_name,
       SUM(b.area_mm2)::numeric / 1000000.0 AS total_meters_squared
FROM bom_line_items b
JOIN materials m ON b.material_id = m.material_id
WHERE b.project_id = :projectId
GROUP BY m.material_name
ORDER BY m.material_name;

If area_mm2 must be computed from length & width (mm):
SELECT m.material_name AS material_name,
       SUM(
         COALESCE(b.area_mm2, COALESCE(b.length_mm,0) * COALESCE(b.width_mm,0))
       )::numeric / 1000000.0 AS total_meters_squared
FROM bom_line_items b
JOIN materials m ON b.material_id = m.material_id
WHERE b.project_id = :projectId
GROUP BY m.material_name
ORDER BY m.material_name;

Postgres example with rounding to 2 decimals:
SELECT m.material_name AS material_name,
       ROUND(SUM(COALESCE(b.area_mm2, b.length_mm*b.width_mm))::numeric / 1000000.0, 2) AS total_meters_squared
FROM bom_line_items b
JOIN materials m ON b.material_id = m.material_id
WHERE b.project_id = $1
GROUP BY m.material_name
ORDER BY m.material_name;

---

## DTO Contract
- MaterialSummaryDTO:
  - materialName: string
  - totalMetersSquared: numeric (scale as per DB or application; may be rounded to 2 decimals by service/UI)
- The service shall return a list of MaterialSummaryDTO entries for the requested project. The list may be empty if there are no bom_line_items for that project.

---

## Test Types & Scope
- Unit tests: validate SQL/projection correctness using an in-memory or test DB and known sample data.
- Integration tests: verify application layer mapping to DTO, parameterization, and that query uses GROUP BY and is scoped to project.
- Contract tests: schema of DTO (exact fields and types).
- Negative/edge tests: null area_mm2, computed area from length/width, zero-area rows, multiple material_ids, very large sums, rounding behavior.
- Performance sanity tests: aggregation on large numbers of line items.
- Security tests: parameterization and prevention of SQL injection.

---

## Structured Test Cases

ID: SQL-CTC-0003-01  
Title: Aggregation returns correct totals per material (precomputed area_mm2)  
Related Requirement: Component Req - 0003 / User Req - 0003  
Priority: High  
Test Type: Integration / DB

Preconditions:
- Test DB populated with bom_line_items for project_id = 42:
  - row1: material_id = M-white, area_mm2 = 600000 (0.6 m^2)
  - row2: material_id = M-white, area_mm2 = 960000 (0.96 m^2)
  - row3: material_id = M-oak,   area_mm2 = 500000 (0.5 m^2)
- materials table contains material_name for each material_id.

Steps:
1. Execute the aggregation SQL/projection with :projectId = 42.
2. Map results to MaterialSummaryDTO list.

Expected Result:
- Returned DTO list contains two entries:
  - { materialName: "<White name>", totalMetersSquared: 1.56 }  // 600000+960000 = 1,560,000 / 1,000,000 = 1.56
  - { materialName: "<Oak name>", totalMetersSquared: 0.50 }
- Values are numeric in m^2. Database calculation matches expected arithmetic.
- Query executed with a parameter for projectId (not string concatenation).

Postconditions:
- Clean up test data.

---

ID: SQL-CTC-0003-02  
Title: Aggregation computes area from length_mm × width_mm when area_mm2 is NULL  
Related Requirement: Robustness / computed area fallback  
Priority: High  
Test Type: Integration / DB

Preconditions:
- Test DB populated for project_id = 100 with two rows for material_id = M-white:
  - row1: area_mm2 = NULL, length_mm = 1000, width_mm = 600
  - row2: area_mm2 = NULL, length_mm = 1200, width_mm = 800

Steps:
1. Run aggregation SQL that uses COALESCE(area_mm2, length_mm*width_mm).
2. Map to DTO.

Expected Result:
- Sum = (1000*600 + 1200*800) = 600,000 + 960,000 = 1,560,000 mm² => 1.56 m².
- DTO returns totalMetersSquared = 1.56.

Postconditions:
- None.

---

ID: SQL-CTC-0003-03  
Title: Aggregation scoped to specific project (no cross-project leakage)  
Related Requirement: Project scoping  
Priority: High  
Test Type: Integration

Preconditions:
- bom_line_items include entries for project_id = 1 and project_id = 2 with same material_ids.

Steps:
1. Execute aggregation for project_id = 1 and separately for project_id = 2.

Expected Result:
- Each result only includes sums for the requested project; values differ if underlying counts differ.
- No combined totals across projects.

Postconditions:
- None.

---

ID: SQL-CTC-0003-04  
Title: DTO contains exactly two fields and types (contract test)  
Related Requirement: DTO contract  
Priority: High  
Test Type: Unit / Contract

Preconditions:
- Aggregation query executes and returns rows.

Steps:
1. Execute the projection that maps directly to DTO or run service method that maps raw rows to DTO.
2. Inspect each returned DTO object for fields and types.

Expected Result:
- Each DTO has exactly materialName (string) and totalMetersSquared (numeric).
- No extra fields are included (e.g., material_id, sum_area_mm2 not exposed).
- Types match contract (e.g., string and numeric/decimal).

Postconditions:
- None.

---

ID: SQL-CTC-0003-05  
Title: Rounding behavior (service vs DB) — verify both raw and rounded outputs  
Related Requirement: UI rounding requirement from User Req - 0003  
Priority: Medium  
Test Type: Integration / UI contract

Preconditions:
- Aggregation raw sum yields a value requiring rounding (e.g., raw total = 1.234567 m²).

Steps:
1. Run SQL that returns raw totalMetersSquared (no rounding).
2. Verify service layer or UI rounds final displayed total to 2 decimals.
3. Optionally run SQL with ROUND(...,2) and compare.

Expected Result:
- DB raw result (e.g., 1.234567) is correct.
- The MaterialSummaryDTO returned to the UI either contains rounded value (1.23) or the UI displays rounded value; test asserts where rounding occurs according to design.
- If DB-side rounding used, database returns 1.23.

Postconditions:
- None.

---

ID: SQL-CTC-0003-06  
Title: Empty project returns empty list (no rows) gracefully  
Related Requirement: Robustness  
Priority: Medium  
Test Type: Integration

Preconditions:
- No bom_line_items for project_id = 9999.

Steps:
1. Run aggregation for project_id = 9999.

Expected Result:
- Query returns zero rows; service returns empty list (not null).
- No exceptions thrown.

Postconditions:
- None.

---

ID: SQL-CTC-0003-07  
Title: Negative or zero area rows are handled (defensive)  
Related Requirement: Data validation / safety  
Priority: Medium  
Test Type: Negative

Preconditions:
- bom_line_items includes rows with area_mm2 <= 0 (e.g., zero or negative due to bad data) for project_id = 55.

Steps:
1. Run aggregation query.
2. Inspect results and logs.

Expected Result:
- Policy options:
  - Default: exclude area_mm2 <= 0 rows from SUM (use WHERE b.area_mm2 > 0 OR COALESCE(... ) > 0) AND log warnings, or
  - Include them if product decides (rare).
- Tests should assert whichever policy the system implements. If excluding, ensure sums do not include negative values and that warnings are emitted.

Postconditions:
- None.

---

ID: SQL-CTC-0003-08  
Title: Performance sanity test: aggregation on large number of rows  
Related Requirement: Performance / scalability  
Priority: Low / Optional  
Test Type: Performance / Integration

Preconditions:
- Test DB seeded with 100k+ bom_line_items across multiple materials for project_id = 777.

Steps:
1. Execute aggregation query and measure execution time and DB resource usage.
2. Ensure proper indexing (e.g., index on project_id, material_id).

Expected Result:
- Aggregation completes within acceptable threshold (define threshold, e.g., < 1s on test hardware).
- Execution plan uses index/efficient scan where appropriate.

Postconditions:
- Tear down seed data.

---

ID: SQL-CTC-0003-09  
Title: SQL injection prevention — parameterized query used  
Related Requirement: Security  
Priority: High  
Test Type: Security / Integration

Preconditions:
- Query executed using parameter binding in the data access layer.

Steps:
1. Review code to ensure prepared statement/parameter binding is used.
2. Optionally pass a malicious projectId string and assert it is treated as parameter, not interpolated SQL.

Expected Result:
- No SQL injection possible; query uses placeholders and bound parameters.
- Tests that inspect SQL logs or query text show parameter markers, not concatenated values.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Material summary SQL aggregation view
  In order to show total sheet area per material for purchasing
  As the backend service
  I want to run a GROUP BY material_id and return materialName and totalMetersSquared

Background:
  Given bom_line_items table contains cut pieces for various projects
  And the materials table maps material_id -> material_name

Scenario: Aggregation returns totals grouped by material for a project
  Given project 42 has White and Oak bom_line_items
  When the material summary query is executed for project 42
  Then the result contains rows:
    | materialName | totalMetersSquared |
    | White Melamine| 1.56              |
    | Oak Veneer    | 0.50              |

Scenario: Aggregation computes area from dimensions when area_mm2 is NULL
  Given a bom_line_item has NULL area_mm2 but length_mm and width_mm present
  When the aggregation runs
  Then area is computed as length_mm × width_mm and included in the SUM

Scenario: Empty project returns empty list
  Given project 9999 has no bom_line_items
  When the aggregation runs
  Then the result is an empty list

Scenario: Aggregation is parameterized to prevent SQL injection
  Given a malicious payload provided as projectId parameter
  When the query runs with parameter binding
  Then the database executes safely and returns no unexpected results

---

## Implementation / Test Automation Guidance

- Place SQL query in a repository layer as a native query or projection (e.g., JPA @Query(nativeQuery=true) returning interface projection, or direct JDBC/NamedParameterJdbcTemplate query).
- Recommended projection mapping:
  - In Java (Spring Data): create interface MaterialSummaryProjection { String getMaterialName(); BigDecimal getTotalMetersSquared(); }
  - Or map ResultSet rows to MaterialSummaryDTO manually.
- Tests:
  - Use a transactional test database (Postgres recommended) with known test fixtures (insert bom_line_items and materials rows), run query, assert returned DTOs.
  - For unit tests where DB is not available, use an in-memory embedded DB that supports required SQL features (H2 with compatibility mode, but confirm numeric/rounding differences).
  - Validate query text coverage if policy requires: assert that SQL contains "GROUP BY", "SUM(" and parameter placeholders rather than string concatenation (inspect logs or capture executed SQL through mock/jdbc spy).
- Indexing note: ensure bom_line_items has an index on project_id and material_id to support efficient GROUP BY queries.
- Rounding policy: decide if DB should return rounded values or if rounding is applied at service/UI layer. Tests should reflect the chosen approach.

Example unit test pseudo-code (Java + Spring JDBC):
  @Test
  void testMaterialSummaryAggregation() {
    insertMaterial("M-white", "White Melamine");
    insertLineItem(projectId=42, materialId="M-white", area_mm2=600000);
    insertLineItem(projectId=42, materialId="M-white", area_mm2=960000);
    List<MaterialSummaryDTO> res = repo.getMaterialSummary(projectId=42);
    assertEquals(1, res.size());
    assertEquals("White Melamine", res.get(0).getMaterialName());
    assertEquals(new BigDecimal("1.56"), res.get(0).getTotalMetersSquared());
  }

---

## Edge Cases & Recommendations
- Decide canonical place for area computation: precompute area_mm2 on insert/update of bom_line_items or compute on-the-fly in aggregation SQL. Precompute simplifies SQL and speeds aggregation; compute-on-read ensures data consistency if dimensions change.
- Clarify rounding responsibilities (DB vs service vs UI) and make tests reflect the decision.
- Include tests for materials with very large total area to ensure no overflow (use numeric/decimal types with sufficient precision).
- Add telemetry to measure query latency for large projects and add threshold-based alerts.
- Preserve traceability: consider returning an optional expanded DTO with contributing line_item_ids for auditing (not required by contract but helpful).
- If project supports multi-tenant DB, ensure WHERE clause constrains both project_id and tenant_id as needed.

---

## Test Data Summary (quick reference)
- Example: project 42:
  - bom_line_items:
    - (project_id=42, material_id=M-white, area_mm2=600000)
    - (project_id=42, material_id=M-white, area_mm2=960000)
    - (project_id=42, material_id=M-oak,   area_mm2=500000)
  - Expected DTO:
    - White Melamine -> 1.56
    - Oak Veneer     -> 0.50

---

What I have provided: a single Markdown test-case document for Component Req - 0003 covering SQL examples, DTO contract, test cases (unit, integration, security, performance), BDD scenarios, and implementation guidance. If you'd like I can now generate the exact SQL view/projection and test skeletons for your preferred database and backend framework — tell me which DB and language/framework to target. 