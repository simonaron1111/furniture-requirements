# Micro Architecture: Furniture & BOM System

## Frontend (Angular)

- Visualizer (/designer): Wraps Three.js. Handles raycasting for edge detection, mesh generation, and 2D plan rendering.

- State Store (/store): Uses NgRx or Signals as the single source of truth. Dispatches actions (e.g., RESIZE_BOX) that trigger API calls and optimistically update the UI.

## Networking / Gateway

- Nginx Proxy: Acts as the reverse proxy between Angular and Spring Boot.

- Handles CORS and SSL termination.

- Routes location /api to the backend upstream.

- Serves static Angular assets for location /.

## Backend (Spring Boot - Hexagonal)

- API Layer: REST Controllers handling HTTP requests and DTO mapping.

- Domain Layer (Core Logic):

- Geometry Engine: Pure Java math. Calculates panel cut lists based on box dimensions and material thickness.

- Rule Engine: Logic gates. Determines hardware needs (e.g., if door > 600mm, add 3 hinges).

- Pricing Service: Aggregates costs: $(Area \times m^2Price) + (Count \times UnitPrice)$.

- Infrastructure Layer: JPA Repositories for Postgres interactions.

## Database (PostgreSQL)

- Relational Schema: Stores Projects (JSON snapshot), Materials (Unit costs), and BOM_Items (Calculated results linked to project).

## Calculation Pipeline

- Input: User updates dimensions in Angular.

- Process: Backend Geometry Engine recalculates panels $\rightarrow$ Rule Engine updates hardware $\rightarrow$ Pricing Service updates totals.

- Output: JSON response updates the 3D View and BOM table simultaneously.

## Quality Assurance (BDD)

- Cucumber: Automated acceptance testing.

- Feature Files: Gherkin syntax (Given/When/Then) mapped to the User Requirements (e.g., BOM_Calculation.feature).

- Glue Code: Java test steps verifying Domain Layer logic.
