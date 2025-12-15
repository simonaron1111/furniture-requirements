# Component Req - 0008 - Component List Creation Engine
## Reference: 
[User Req - 0007 - Component List Retrieval and Management](./../../user-requirements/0007%20-%20Component%20List%20Retrieval%20and%20Management.md)

## Requirement
- Given a FurnitureBody entity exists with defined geometric properties and front elements containing raw material specifications.
- When the createFromFurnitureBody() method is called with valid furniture body ID and creator information.
- Then the service must analyze the furniture geometry, extract raw material requirements from front elements, calculate total quantities for identical materials, and persist a new ComponentList entity with associated RawMaterial entries in a single atomic transaction.

## Acceptance Criteria
- The creation process must validate furniture body existence before proceeding.
- Raw materials with identical dimensions and material types must be consolidated with cumulative quantities.
- The service must handle null or empty raw material lists gracefully without failing.
- All database operations must occur within a single transaction boundary.
- The created component list must include audit fields (created_at, created_by) with accurate timestamps and user information.