# Component Req - 0007 - Component List Data Access Layer
## Reference: 
[User Req - 0007 - Component List Retrieval and Management](./../../user-requirements/0007%20-%20Component%20List%20Retrieval%20and%20Management.md)

## Requirement
- Given the ComponentListService receives a request for component list data with furniture body ID and optional filtering parameters.
- When the getComponentListsByFurnitureBodyId() method is invoked.
- Then the service must query the repository layer using JPA specifications to retrieve component lists with their associated raw materials and manufactured components, applying eager loading for optimal performance and returning ComponentListDTO objects with complete material hierarchies.

## Acceptance Criteria
- The service must use @Transactional annotation to ensure data consistency during complex queries.
- Raw materials must be fetched with their material type information in a single query to avoid N+1 problems.
- Manufactured components must include their type definitions and quantity calculations.
- The service must handle pagination for large component lists (page size configurable, default 20 items).
- Repository queries must use indexed database fields (furniture_body_id, created_at) for optimal performance.