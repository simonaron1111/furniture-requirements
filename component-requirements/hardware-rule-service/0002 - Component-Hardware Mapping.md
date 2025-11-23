# Component Req - 0002 - Component-Hardware Mapping
## Reference: 
[User Req - 0002 - Hardware Association Logic](./../../user-requirements/0002%20-%20Hardware%20Association%20Logic.md)
## Requirement
- Given a set of Component objects (Doors, Drawers) that have been successfully placed in the model.
- When the applyHardwareRules() method scans these components.
- Then the service must lookup the HardwareMappingConfiguration (a JSON or Map structure) to determine dependencies (e.g., Drawer $\rightarrow$ SlidePair + Handle) and inject these derivative items into the BOM stream.

## Acceptance Criteria
- The service must lookup the HardwareMappingConfiguration (a JSON or Map structure) to determine dependencies (e.g., Drawer $\rightarrow$ SlidePair + Handle) and inject these derivative items into the BOM stream.