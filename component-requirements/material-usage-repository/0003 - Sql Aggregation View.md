# Component Req - 0003 - SQL Aggregation View
## Reference: 
[User Req - 0003 - Material Surface Area Aggregation](../../user-requirements/0003 - Material Surface Area Aggregation.md)
## Requirement
- Given the bom_line_items table is populated with individual cut pieces for a project.
- When the application requests the material summary.
- Then a native SQL query or Projection must run a GROUP BY material_id operation to sum the area_mm2 and divide by $1,000,000$, returning a streamlined Data Transfer Object (DTO) containing only MaterialName and TotalMetersSquared.