# TC-0008 - Raw Material Inventory Test Cases

## Test Case 1: Search Raw Materials by Material Type
**Given:** Raw materials exist for "Oak Veneer" (ID=1) and "White Melamine" (ID=2)
**When:** GET /api/raw-materials?materialTypeId=1 is called
**Then:**
- Response status is 200 OK
- Response contains only "Oak Veneer" materials
- Each material includes dimensions, quantity, and last updated timestamp
- Materials are sorted by creation date descending

## Test Case 2: Find Raw Material by Exact Dimensions
**Given:** A raw material exists with dimensions 1200mm × 600mm × 18mm
**When:** GET /api/raw-materials/find?length=1200&width=600&height=18&materialTypeId=1
**Then:**
- Response status is 200 OK
- Response contains the exact matching material
- Material details include current stock quantity and material type information

## Test Case 3: Raw Material Not Found by Dimensions
**Given:** No raw material exists with dimensions 5000mm × 3000mm × 25mm
**When:** GET /api/raw-materials/find?length=5000&width=3000&height=25&materialTypeId=1
**Then:**
- Response status is 404 Not Found
- Response includes descriptive error message
- Error message suggests alternative dimensions or material types

## Test Case 4: Get Material Type Summary
**Given:** Multiple raw materials exist for different material types
**When:** GET /api/raw-materials/summary is called
**Then:**
- Response status is 200 OK
- Response contains aggregated data by material type
- Each summary includes total quantity, surface area, and average price
- Surface area calculations are accurate (length × width × quantity)