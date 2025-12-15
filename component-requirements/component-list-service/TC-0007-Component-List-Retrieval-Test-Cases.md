# TC-0007 - Component List Retrieval Test Cases

## Test Case 1: Retrieve Component Lists by Furniture Body ID
**Given:** A furniture body exists with ID=1 and has 2 associated component lists
**When:** GET /api/component-lists?furnitureBodyId=1 is called
**Then:** 
- Response status is 200 OK
- Response contains array of 2 component list objects
- Each component list includes raw materials and manufactured components
- Raw materials include dimensions (length, width, height) and quantities
- Manufactured components include type name and required quantity

## Test Case 2: Create Component List from Furniture Body
**Given:** A furniture body exists with ID=5 containing front elements with raw materials
**When:** POST /api/component-lists/from-furniture/5 with {"createdBy": 123}
**Then:**
- Response status is 201 Created
- New component list is created in database
- Raw materials are extracted from furniture body front elements
- Identical materials are consolidated with cumulative quantities
- Response includes complete component list with generated ID

## Test Case 3: Handle Non-existent Furniture Body
**Given:** No furniture body exists with ID=999
**When:** GET /api/component-lists?furnitureBodyId=999 is called
**Then:**
- Response status is 200 OK
- Response contains empty array []
- No server errors are logged

## Test Case 4: Handle Invalid Component List Creation
**Given:** A furniture body with ID=10 has no raw materials defined
**When:** POST /api/component-lists/from-furniture/10 with {"createdBy": 123}
**Then:**
- Response status is 201 Created
- Component list is created with empty raw materials list
- Created timestamp and creator are properly set