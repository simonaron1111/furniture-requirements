# User Req - 0001 - Automatic Panel Dimension Calculation
## Requirement description
The system must automatically calculate the list of required wooden panels (cut list) based on the geometric dimensions of the furniture drawn in the 2D/3D view, accounting for material thickness.

## Requirement
- Given a furniture project exists with a defined outer "Box" of dimensions $2000mm (H) \times 1000mm (W) \times 600mm (D)$ and a material thickness of $18mm$.
- When the user navigates to the "Bill of Materials" tab.
- Then the panel list should populate with specific parts (e.g., 2x Side Panels, 1x Top Panel, 1x Bottom Panel) showing calculated dimensions (e.g., Top Panel: $1000mm \times 600mm$).

## Acceptance Criteria
- The sum of internal panel dimensions + thickness must equal the total outer dimensions.
- Changes to the 3D model dimensions must trigger a recalculation of the BOM upon refresh.
- Dimensions must be displayed in millimeters (mm).