# Component Req - 0004 - Raycast Edge Detection
## Reference: 
[User Req - 0004 - Edge Banding Calculation](./../../user-requirements/0004%20-%20Edge%20Banding%20Calculation.md)
## Requirement
- Given a 3D mesh of a panel rendered in the Angular view.
- When the user performs a mouse click on a specific face of the mesh (raycast intersection).
- Then the interaction service must identify the specific edge_index (0-3) of the underlying data model corresponding to that 3D face and emit an EdgeSelected event containing the Panel ID and the specific side clicked.