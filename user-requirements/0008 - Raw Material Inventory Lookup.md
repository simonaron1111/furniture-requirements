# User Req - 0008 - Raw Material Inventory Lookup
## Requirement description
The system must provide users with the ability to search and view raw material inventory, including specifications, dimensions, and availability status to support production planning and material procurement.

## Requirement
- Given raw materials exist in the system with defined types, dimensions, and quantities.
- When the user accesses the "Raw Materials" inventory section.
- Then the system displays a searchable list of all raw materials with their material type, available dimensions (length × width × height), current stock quantity, and last updated timestamp.

## Acceptance Criteria
- Raw materials must be filterable by material type (e.g., "Oak Veneer", "White Melamine").
- Search functionality must allow users to find materials by specific dimensions.
- Quantities must be displayed with appropriate units (pieces, square meters, linear meters).
- The system must show when each material entry was last updated.
- Users can view detailed specifications for each raw material type including density, price per unit, and surface finish properties.