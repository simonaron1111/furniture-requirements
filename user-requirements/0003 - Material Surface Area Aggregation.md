# User Req - 0003 - Material Surface Area Aggregation
## Requirement description
The system must aggregate the total square meterage ($m^2$) required for each specific material type (e.g., White Melamine vs. Oak Veneer) to assist in purchasing sheets.
## Requirement
- Given the design uses "White Melamine" for the internal carcass and "Oak Veneer" for the door fronts.
- When the user views the BOM Summary section.
- Then the system displays two distinct totals: Total Area for White Melamine and Total Area for Oak Veneer.

## Acceptance criteria
- Area is calculated as $Length \times Width$ for every panel.
- Items with different material_id are not summed together.
- The total is rounded to two decimal places.