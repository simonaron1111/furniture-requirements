# User Req - 0009 - Component List Material Optimization
## Requirement description
The system must automatically optimize raw material usage by suggesting the most efficient material cutting patterns and identifying opportunities to reuse materials across multiple components within a component list.

## Requirement
- Given a component list contains multiple raw material requirements of the same material type with varying dimensions.
- When the user requests material optimization for the component list.
- Then the system analyzes all required pieces and suggests optimized cutting layouts that minimize material waste and calculates the total material sheets needed.

## Acceptance Criteria
- The system must group materials by type and suggest standard sheet sizes that can accommodate multiple pieces.
- Waste percentage must be calculated and displayed for each material type.
- Alternative material arrangements must be suggested when waste exceeds 15%.
- The optimization must account for material grain direction and cutting blade thickness (kerf).
- Users can accept or modify the suggested optimization before finalizing the component list.