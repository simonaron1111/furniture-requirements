# Component Req - 0006 - Optimistic UI Updates
## Reference: 
[User Req - 0006 - Manual BOM Entries](../../user-requirements/0006 - Manual BOM Entries.md)

## Requirement
- Given the user has submitted the "Add Custom Item" form in the UI.

- When the form submission action is dispatched.

- Then the Store must immediately update the local bomList$ observable array to display the new item (Optimistic Update) while simultaneously triggering the API call postCustomItem() in the background to ensure the UI feels instant.