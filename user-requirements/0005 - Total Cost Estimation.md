# User Req - 0005 - Total Cost Estimation
## Requirement description
The system must calculate a total estimated cost for the project by multiplying the quantity/area of items in the BOM against a stored unit price in the database.

## Requirement
- Given the database contains a price of $20 per $m^2$ for wood and $5 per unit for hinges.
- When the BOM is rendered.
- Then a "Total Cost" column should display the line item cost, and a "Grand Total" footer should display the sum of all line items.

## Acceptance criteria
- Formula: $(Area \times Price_{m2}) + (Quantity \times Price_{unit})$.
- If a price is missing (null), the system warns the user rather than showing $0.00.
- Currency symbols are displayed consistent with user locale.