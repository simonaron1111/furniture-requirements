# Component Req - 0005 - Composite Pricing Calculator
## Reference: 
[User Req - 0005 - Total Cost Estimation](../../user-requirements/0005 - Total Cost Estimation.md)
## Requirement
- Given a populated BOM object containing both Panel items (measured in area) and Hardware items (measured in units).
- When the calculateProjectCost() method is triggered.
- Then the service must inject the PricingStrategy to fetch current unit costs from the DB, apply the math $(Qty \times UnitPrice)$ OR $(Area \times m^2Price)$ respectively, and reduce the stream to a single BigDecimal total.