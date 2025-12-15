# Component Req - 0010 - Material Type Aggregation Service
## Reference: 
[User Req - 0008 - Raw Material Inventory Lookup](./../../user-requirements/0008%20-%20Raw%20Material%20Inventory%20Lookup.md)

## Requirement
- Given multiple RawMaterial entries exist with the same RawMaterialType but different dimensions and quantities.
- When the getMaterialSummaryByType() method is called on the RawMaterialService.
- Then the service must execute a GROUP BY query on raw_material_type_id, calculate total quantities and surface areas per material type, and return a List&lt;MaterialSummaryDTO&gt; containing material type name, total quantity, total surface area in square meters, and average price per unit.

## Acceptance Criteria
- Surface area calculation must use the formula: $(length \times width) \times quantity$ for each material entry.
- Totals must be aggregated by material type ID to consolidate identical materials with different dimensions.
- Price calculations must account for current market rates from the material_types table.
- The service must handle division by zero gracefully when calculating averages for materials with zero quantity.
- Results must be sorted by material type name alphabetically for consistent user experience.