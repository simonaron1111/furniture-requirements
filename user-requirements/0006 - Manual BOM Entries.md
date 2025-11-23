# User Req - 0006 - Manual BOM Entries (Consumables)
## Requirement description
The user must be able to manually add items to the generated BOM for consumables that are not geometrically drawn (e.g., Glue, Dowels, Feet).

## Requirement
- Given the automatically generated BOM is displayed.

- When the user clicks "Add Custom Item", fills in "Wood Glue", Quantity "1", and Price "5.00".

- Then this item is persisted in the database linked to this specific project and included in the Grand Total.

## Acceptance criteria

- Manual entries are visually distinct from auto-generated entries (e.g., different icon or highlight).

- Manual entries persist even if the 3D model is resized and the BOM auto-recalculates.