# Component Req - 0009 - Raw Material Query Repository
## Reference: 
[User Req - 0008 - Raw Material Inventory Lookup](./../../user-requirements/0008%20-%20Raw%20Material%20Inventory%20Lookup.md)

## Requirement
- Given the RawMaterialRepository interface extends JpaRepository and includes custom query methods.
- When the findByRawMaterialTypeAndDimensions() method is invoked with material type ID and dimension parameters.
- Then the repository must execute a native SQL query that joins raw_materials and raw_material_types tables, applying exact dimension matching (length, width, height) and return Optional&lt;RawMaterial&gt; to handle cases where no matching material exists in inventory.

## Acceptance Criteria
- The repository must use @Query annotation with native SQL for optimal performance on indexed columns.
- Query parameters must be properly sanitized to prevent SQL injection attacks.
- The method must return Optional&lt;RawMaterial&gt; to handle null cases gracefully.
- Database indexes must exist on (raw_material_type_id, length, width, height) for sub-millisecond query performance.
- The repository must support batch operations for bulk material lookups during component list processing.