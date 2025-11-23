# User Req - 0004 - Edge Banding Calculation
## Requirement description
The system must calculate the linear meters of edge banding required, distinguishing between visible edges (requiring premium banding) and non-visible edges (requiring standard or no banding).

## Requirement
- Given a shelf panel where only the front-facing edge is flagged as "Visible".

- When the BOM calculates the "Edge Banding" category.

- Then the length of the front edge is added to the "Premium Edging" total, and the other three sides are added to "Standard Edging" or ignored based on configuration.

## Acceptance criteria

- Linear meters are summed correctly per material type.

- The UI allows the user to toggle which edges of a rect are "taped/banded" in the design view, reflecting immediately in the BOM.