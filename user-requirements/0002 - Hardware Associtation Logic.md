# User Req - 0002 - Hardware Association Logic
## Requirement description
The system must automatically add specific hardware to the BOM based on the components added to the furniture design (e.g., adding a door implies adding hinges; adding a drawer implies adding slides).
## Requirement
- Given the user has added a "Door" component to the front of the furniture box.
- When the BOM is generated.
- Then the list must include "Standard Hinges" and "Door Handle" with the correct quantities (e.g., 2 hinges per door up to 1000mm height).
## Acceptance criteria
- Door adds $\ge 2$ hinges (logic based on door height) and 1 handle.
- 1 Drawer adds 1 pair of drawer slides and 1 handle.
- Deleting the component from the 3D view removes the associated hardware from the BOM.