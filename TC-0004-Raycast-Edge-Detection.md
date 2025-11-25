# Test Case Document — Component Req 0004: Raycast Edge Detection

## Reference
User Req - 0004 - Edge Banding Calculation  
(See: ../../user-requirements/0004 - Edge Banding Calculation.md)

## Component Requirement
Given a 3D mesh of a panel rendered in the Angular view, when the user performs a mouse click on a specific face of the mesh (raycast intersection), then the interaction service must identify the specific edge_index (0-3) of the underlying data model corresponding to that 3D face and emit an EdgeSelected event containing the Panel ID and the specific side clicked.

Acceptance Criteria:
- Interaction service emits an EdgeSelected event on raycast selection with payload containing Panel ID and edge_index in {0,1,2,3}.
- Mapping from clicked mesh face (or triangle) to model edge_index must be deterministic and stable regardless of mesh triangulation.
- No database or remote calls performed during the selection; detection is purely client-side.

---

## Assumptions (explicit)
- Panels are rectangular quads with four logical sides indexed 0..3 in the data model. The canonical mapping of indices is:
  - edge_index = 0 => front edge
  - edge_index = 1 => right edge
  - edge_index = 2 => back edge
  - edge_index = 3 => left edge
  (If your domain uses a different naming/order, replace with project convention; tests are parametric.)
- The 3D mesh representing a panel may be made of several triangular faces; each logical side (face in the user model) corresponds to one or more triangular mesh faces. The mesh contains metadata linking triangles to panelId and logicalSide (e.g., object.userData.panelId and face.userData.side OR materialIndex/per-face attribute).
- Raycasting is implemented with Three.js Raycaster (or equivalent). The raycast intersection result includes:
  - object (mesh)
  - face (triangle) or faceIndex
  - point (world coordinates)
  - uv / barycentric (optional)
- The InteractionService receives normalized device coordinates from UI click, performs raycast, resolves intersection, maps triangle/face -> logical side -> edge_index, and emits EdgeSelected event:
  - EdgeSelected payload example: { panelId: "panel-123", edgeIndex: 0, intersectionPoint: {x,y,z}, source: "mouse" }
- Clicks near vertices or on shared edges are resolved deterministically (policy described in tests).
- Backface clicks: selection policy may be "select both sides" or "ignore backfaces" — tests include both behaviors depending on configuration.
- Selection should be debounced to avoid duplicate events when multiple intersection results are returned for same click.

---

## Key mapping rules to test
- Triangle-to-side mapping: any triangle that belongs to a logical side must map to the same edge_index as that side.
- Per-triangle metadata (preferred): face.userData.side (0..3) or mesh.geometry.groups/materialIndex referencing side index. If metadata is absent, mapping must be derivable from barycentric/UV or position orientation with respect to the panel local axes.
- Deterministic tie-break: when click intersects an edge or vertex shared by multiple sides, selection chooses based on:
  - Primary: nearest triangle/intersection distance to camera
  - Secondary: prefer visible side (normal pointing toward camera)
  - Tertiary: configured tie-break rule (e.g., prefer smaller edge index)
  Tests will assert chosen tie-break behavior.

---

## Test Types and Scope
- Unit tests for InteractionService and Face->Edge mapping helper functions (pure logic, mocking raycast results).
- Integration tests within Angular component + Three.js scene using test harness that can simulate raycast hits or run headless WebGL where feasible.
- E2E tests using Playwright (or Cypress with WebGL support) to simulate clicks on the canvas and assert emitted events or UI effect.
- Negative/Edge tests: clicks on background (no intersection), clicks on overlapping panels, clicks near vertex, backface clicks, extremely small panels, high-DPI coordinate transforms, rotated/scaled panels.
- Contract tests: EdgeSelected event schema, immutability, no DB calls.
- Performance: ensure selection latency is small (< 100ms typical), and clicking repeatedly does not leak events.

---

## Structured Test Cases

ID: REQ-CTC-0004-01  
Title: Raycast maps triangle face to edge_index and emits EdgeSelected (happy path)  
Related Requirement: Component Req - 0004  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Panel mesh exists and has a logical side mapping (mesh.userData.panelId = "panel-1"; face/triangle metadata maps to side 0).
- InteractionService is running in test harness.
- Raycast result is simulated/mocked to return intersection with triangle tagged side = 0.

Steps:
1. Simulate a click on the canvas at coordinates corresponding to intersection with triangle for side 0 (or mock Raycaster to return intersection with face.userData.side = 0).
2. InteractionService handles click and processes raycast result.

Expected Result:
- EdgeSelected event emitted exactly once.
- Event payload equals: { panelId: "panel-1", edgeIndex: 0, intersectionPoint: {...} }.
- No repository/DB calls made (assert no network calls).

Postconditions:
- None.

---

ID: REQ-CTC-0004-02  
Title: Triangulated side with multiple triangles maps consistently to same edge_index  
Related Requirement: Deterministic mapping across triangles  
Priority: High  
Test Type: Unit

Preconditions:
- Logical side 1 is represented by N triangles (faces). Each face has metadata side = 1.

Test Data:
- Mock intersections for each triangle of side 1.

Steps:
1. For each triangle of side 1, simulate an intersection and trigger InteractionService.
2. Capture EdgeSelected event for each simulation.

Expected Result:
- For every triangle intersection, emitted edgeIndex == 1 and panelId is correct.
- No variation depending on which triangle hit.

Postconditions:
- None.

---

ID: REQ-CTC-0004-03  
Title: Click on edge/vertex shared by two sides uses deterministic tie-break rule (nearest visible)  
Related Requirement: Tie-break behavior for ambiguous clicks  
Priority: High  
Test Type: Unit / Integration

Preconditions:
- Two adjacent sides (side 0 and side 1) share an edge; click simulated exactly on the shared edge/vertex.
- Both triangles' intersections return same distance or near-equal distances. Camera is positioned to make one triangle's normal point toward the camera.

Steps:
1. Simulate raycast intersection that returns a tie between faces for side 0 and side 1.
2. Invoke InteractionService.

Expected Result:
- InteractionService selects triangle whose normal faces the camera (visible side) OR nearest triangle by intersection distance according to defined tie-break policy.
- EdgeSelected emitted with the chosen edgeIndex.
- Tests assert the chosen rule; adjust expected behavior if policy differs.

Postconditions:
- None.

---

ID: REQ-CTC-0004-04  
Title: Backface handling — when backface selection disabled, clicks on backface ignored  
Related Requirement: Backface culling policy  
Priority: Medium  
Test Type: Unit / Integration

Preconditions:
- Configuration: allowBackfaceSelection = false.
- Simulated intersection is on triangle whose normal points away from camera.

Steps:
1. Simulate click resulting in an intersection where face.normal · viewDirection > 0 (backface).
2. InteractionService processes raycast.

Expected Result:
- No EdgeSelected event emitted (selection ignored).
- If UI should show a "no selection" indicator, verify that.
- When allowBackfaceSelection = true, same intersection should emit EdgeSelected.

Postconditions:
- Reset config.

---

ID: REQ-CTC-0004-05  
Title: Angular view coordinate transform correctness — click-to-world conversion yields same faceIndex as direct raycast mock  
Related Requirement: Screen->NDC->Ray conversion correctness  
Priority: High  
Test Type: Integration / UI

Preconditions:
- Running test harness where canvas dimensions and devicePixelRatio can be controlled.
- Known camera transform and mesh location.

Steps:
1. Compute screen coordinates for a known face center given camera, renderer, and canvas metrics.
2. Use the real raycast function (not mocked) to perform intersection from those screen coordinates.
3. Verify the resulting faceIndex and mapped edgeIndex match expected value.

Expected Result:
- Real raycast selects the expected triangle and InteractionService emits EdgeSelected with correct panelId and edgeIndex.
- Test repeated with different devicePixelRatio and browser sizes to assert robustness.

Postconditions:
- None.

---

ID: REQ-CTC-0004-06  
Title: EdgeSelected event schema validation (contract)  
Related Requirement: Event payload contract  
Priority: High  
Test Type: Unit / Contract

Preconditions:
- InteractionService emits events to an EventBus or RxJS Subject which tests can subscribe to.

Steps:
1. Simulate a selection that triggers EdgeSelected.
2. Capture the emitted event.

Expected Result:
- Payload contains:
  - panelId: string (non-empty)
  - edgeIndex: integer in [0,1,2,3]
  - intersectionPoint: {x:number,y:number,z:number} (optional but recommended)
  - source: string (e.g., "mouse" or "touch")
- No extra unexpected fields present (or extras documented).
- Event emitted only once per user click.

Postconditions:
- None.

---

ID: REQ-CTC-0004-07  
Title: No intersection click (background) emits no EdgeSelected event and may emit SelectionCleared  
Related Requirement: Negative case / clearing selection  
Priority: Medium  
Test Type: Unit

Preconditions:
- Click at a screen coordinate with no raycast intersections.

Steps:
1. Simulate click on empty space.
2. Process with InteractionService.

Expected Result:
- No EdgeSelected emitted.
- If UI contract defines SelectionCleared event, ensure it is emitted exactly once.
- No exceptions thrown.

Postconditions:
- None.

---

ID: REQ-CTC-0004-08  
Title: Multiple overlapping panels — top-most visible panel is selected (z-order)  
Related Requirement: Selection priority among overlapping mesh objects  
Priority: High  
Test Type: Integration / E2E

Preconditions:
- Two panels overlap from camera perspective; panel A visually in front of panel B.

Steps:
1. Click at coordinate overlapping both panels.
2. InteractionService raycasts and uses nearest intersection to camera.

Expected Result:
- EdgeSelected event corresponds to panel A (the top-most / nearest intersection).
- PanelId in payload equals panel A id.

Postconditions:
- None.

---

ID: REQ-CTC-0004-09  
Title: Debounce/duplicate prevention — a single click yields a single event even if raycast returns multiple intersections in same frame  
Related Requirement: Event deduplication  
Priority: Medium  
Test Type: Unit / Integration

Preconditions:
- Scene returns multiple intersections (e.g., click passes through thin geometry), or input system emits duplicate pointerdown events.

Steps:
1. Trigger a single user click (or mock a single pointer event) but configure raycast to return multiple intersections including the same panel triangles.
2. InteractionService handles intersections.

Expected Result:
- InteractionService emits one EdgeSelected event for the chosen triangle/edge.
- No duplicate events emitted for same click.

Postconditions:
- None.

---

ID: REQ-CTC-0004-10  
Title: Robustness: invalid/missing triangle metadata falls back to geometric side detection (orientation/UV) or returns validation warning  
Related Requirement: Defensive behavior for imperfect meshes  
Priority: Medium  
Test Type: Unit

Preconditions:
- Triangle or mesh lacks explicit mapping metadata.

Steps:
1. Simulate an intersection on a triangle without face.userData.side and without materialIndex mapping.
2. InteractionService must attempt fallback:
   - Option A: compute local face normal and compare to panel local axes to deduce side
   - Option B: use UV coordinate mapping to identify side
   - Option C: return no-selection with logged warning (depending on product policy)

Expected Result:
- If fallback implemented: deduced edgeIndex emitted and consistent with expected side.
- If fallback not implemented: no selection; a clear warning/log entry generated describing missing mapping.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Raycast Edge Detection and EdgeSelected event
  In order to allow users to toggle edge banding by clicking edges in 3D view
  As an interaction service in Angular/Three.js
  I want to translate a mouse click on a mesh face into an EdgeSelected event containing panelId and edgeIndex

Background:
  Given a 3D panel mesh is rendered in the scene with triangles mapped to logical sides (0..3)

Scenario: Click on a face mapped to front edge emits EdgeSelected with edgeIndex 0
  Given triangle T belongs to panel "panel-1" and side 0
  When I click on the mesh over triangle T
  Then an EdgeSelected event shall be emitted with { panelId: "panel-1", edgeIndex: 0 }

Scenario: Click on shared edge resolves to visible side
  Given triangle T1 (side 0) and T2 (side 1) share the same edge
  And the camera faces T1
  When I click on the shared edge
  Then the EdgeSelected event shall contain the edgeIndex of T1 (visible side)

Scenario: Click on empty space emits no EdgeSelected
  Given I click on a blank area of the canvas (no intersections)
  Then no EdgeSelected event shall be emitted
  And SelectionCleared event may be emitted if defined

Scenario: Missing triangle metadata triggers fallback or warning
  Given a triangle lacks userData.side
  When I click on that triangle
  Then the service shall attempt to deduce side from geometry and emit EdgeSelected, or emit a warning and not select

---

## Implementation / Test Automation Guidance

Unit test guidance:
- Target: InteractionService and helper functions (mapFaceToEdge, resolveTieBreak, computeSideFromFace).
- Use Jasmine/Karma (Angular) or Jest for unit tests.
- Mock Raycaster intersection results to feed controlled intersection objects:
  - { object: meshMock, faceIndex: n, face: { a,b,c, normal }, point: {x,y,z}, distance }
- Provide meshMock with geometry that includes:
  - geometry.faces or indexedBufferGeometry with groups and per-face attributes
  - meshMock.userData.panelId and a mapping from faceIndex -> side (faceSideMap)
- Assert subscription to service.edgeSelected$ emits correct payload.
- Assert no HTTP or DB calls are made (spy on HttpClient or repository and expect zero calls).

Sample unit test pseudo-code (Jasmine-like):
  it('emits EdgeSelected for face mapped to side 0', () => {
    const mesh = createMeshMock('panel-1', faceSideMap = { 42: 0 });
    const intersection = { object: mesh, faceIndex: 42, point: {x:0,y:0,z:0}, distance: 10 };
    raycasterMock.intersectObjects.and.returnValue([intersection]);
    interactionService.handleClick(screenX, screenY);
    expect(edgeSelectedSpy).toHaveBeenCalledWith({ panelId: 'panel-1', edgeIndex: 0, intersectionPoint: jasmine.any(Object) });
  });

Integration / Angular component guidance:
- Use TestBed to create component and inject mocked renderer/camera or use a headless WebGL context if available.
- Where possible, test screen->NDC->ray conversion by computing the expected screen coordinates of face centroids using camera.project and then simulating pointer events at those coordinates.

E2E guidance (Playwright recommended for WebGL support):
- Launch the app in a real browser context with a deterministic scene seeded (test project).
- Use page.mouse.click(x,y) on the canvas at coordinates that correspond to known face centers (precomputed or derived from a test-only helper endpoint exposing face screen positions).
- Listen for EdgeSelected events via a test hook (e.g., window.__test_hooks__ capturing events) or assert the UI changed (edge flag toggled in DOM).
- Ensure tests wait for animation frames and any debounce that InteractionService uses.

Edge metadata recommendations:
- Prefer attaching explicit side mapping metadata to faces/geometry groups at mesh creation time (e.g., set geometry.groups or BufferAttribute "sideIndex" or set materialIndex per side). This simplifies mapping and testing.
- If using BufferGeometry, add an Int8BufferAttribute named "sideIndex" indexed per-face (or per-vertex) to map triangles to logical side.

Event contract:
- Define a TypeScript interface:
  interface EdgeSelectedEvent { panelId: string; edgeIndex: number; intersectionPoint?: {x:number,y:number,z:number}; source?: 'mouse'|'touch'|'keyboard'; }
- Unit tests should import and assert event shape.

---

## Edge Cases & Additional Recommendations
- High-DPI and canvas scaling: tests must cover devicePixelRatio and CSS scaling to ensure correct screen -> NDC conversion.
- Rotated/scaled panels: mapping must use mesh local coordinates to compute side, tests should include rotated meshes.
- Mesh LOD / dynamic re-meshing: verify that face->side metadata is preserved after re-mesh or geometry updates; add tests for reloaded meshes.
- Accessibility: provide keyboard selection alternative (focus + arrow keys to choose edge); include tests if this is in scope.
- Telemetry: consider logging selection counts and durations for performance monitoring; unit tests may assert telemetry calls are made if instrumentation is required.
- Tie-break rules must be documented and stable; update tests when policy changes.
- Security: ensure no sensitive data is emitted in event payload.
- If selection affects server state (e.g., toggling banding flags persisted), test the subsequent save flows and ensure the initial selection logic remains client-only.

---

## Test Data Summary (quick reference)
- panelId: "panel-1"
- faceIndex -> side map: { 0:0, 1:0, 2:1, 3:1, 4:2, 5:2, 6:3, 7:3 }  // example for two-triangle-per-side mesh
- Expected EdgeSelected payload: { panelId: "panel-1", edgeIndex: N, intersectionPoint: {x,y,z}, source: "mouse" }

---

What I produced: a comprehensive Markdown test-case document for Raycast Edge Detection including unit, integration and E2E tests, BDD scenarios, payload contract, tie-break rules, and automation suggestions for Angular/Three.js.

What's next: I can generate runnable unit-test skeletons for your InteractionService in TypeScript (Jasmine/Karma or Jest), or produce a Playwright E2E test that clicks the canvas and asserts the EdgeSelected behavior — tell me which output and target test framework you prefer and I will produce the code next.