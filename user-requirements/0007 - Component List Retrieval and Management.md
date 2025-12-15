# User Req - 0007 - Component List Retrieval and Management
## Requirement description
The system must allow users to view, create, and manage component lists for furniture projects, providing a comprehensive overview of all required raw materials and manufactured components with their quantities.

## Requirement
- Given a furniture project exists with defined geometric dimensions and material specifications.
- When the user navigates to the "Component Lists" section and selects a specific furniture body.
- Then the system displays the complete component list showing all raw materials (with dimensions and quantities) and manufactured components (with types and quantities) required for the project.

## Acceptance Criteria
- Component lists must display raw materials grouped by material type with individual dimensions (length, width, height) in millimeters.
- Manufactured components must show the component type name and required quantity.
- Users can create new component lists from existing furniture body designs.
- The creation timestamp and creator information must be visible for each component list.
- Component lists must be sortable by creation date and filterable by furniture body.