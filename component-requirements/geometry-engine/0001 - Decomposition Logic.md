# Component: Backend Domain Layer (GeometryEngine)
## Reference: 
[User Req - 0001 - Automatic Panel Dimension Calculation](./../../user-requirements/0001%20-%20Automatic%20Panel%20Dimension%20Calculation.md)
## Requirement
- Given a generic FurnitureBox entity defined by vector dimensions $(x, y, z)$ and a MaterialStrategy indicating construction method (e.g., "Sides-Surround-Top").
- When the decomposeToParts() method is invoked.
- Then the engine must return a list of PartDefinition objects where the geometric subtraction of material thickness is applied purely based on the math of the construction strategy (e.g., Top Width = Box Width - $(2 \times Thickness)$), without accessing the database.

## Acceptance Criteria
- The engine must return a list of PartDefinition objects where the geometric subtraction of material thickness is applied purely based on the math of the construction strategy (e.g., Top Width = Box Width - $(2 \times Thickness)$), without accessing the database.