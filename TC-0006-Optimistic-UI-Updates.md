# Test Case Document — Component Req 0006: Optimistic UI Updates

## Reference
User Req - 0006 - Manual BOM Entries (Consumables)  
(See: ../../user-requirements/0006 - Manual BOM Entries (Consumables).md)

## Component Requirement
Given the user has submitted the "Add Custom Item" form in the UI, when the form submission action is dispatched, then the Store must immediately update the local bomList$ observable array to display the new item (Optimistic Update) while simultaneously triggering the API call postCustomItem() in the background to ensure the UI feels instant.

Acceptance Criteria:
- The Store updates UI immediately on submit (optimistic).
- postCustomItem() is invoked in background with correct payload.
- On API success, the optimistic item is reconciled with server response (server id, persisted fields).
- On API failure, the optimistic change is reverted (or marked failed) and the user is notified; behavior should follow configured rollback policy.
- No duplicate visual items remain after successful reconciliation; temporary client-only IDs used for optimistic items are replaced by server IDs.

---

## Assumptions (explicit)
- State management: application uses a Store pattern (NgRx/Redux/Vuex/MobX). Tests can be adapted to chosen framework.
- Action flow:
  - UI dispatches AddManualItemOptimistic({ tempItem }) where tempItem contains a client-generated tempId (UUID prefixed with "tmp-"), name, qty, unitPrice, isManual = true.
  - Effect/middleware triggers postCustomItem(tempItemPayload) and listens for success/failure actions (AddManualItemSuccess / AddManualItemFailure).
  - On success, reducer replaces temp item with server item (serverId) or patches fields (created_at, createdBy).
  - On failure, reducer either removes the temp item or marks it as failed (isFailed = true) depending on rollback policy.
- Pricing and Grand Total recalculation run after optimistic insert; Grand Total may display provisional amount (indicated) until server confirms.
- API contract: POST /api/projects/{projectId}/bom/manual-items returns full persisted item with id and server metadata.
- Temporary ID approach: client assigns tempId; server returns permanent id; mapping performed in success handler.
- Dedup / idempotency: client uses idempotency-key or tempId to prevent duplicate server inserts on retries; tests will verify idempotency behavior.
- Rollback policy is configurable; tests include both "revert on failure" and "mark failed item" variants — state the chosen policy in each test.

---

## Test Types & Scope
- Unit tests: reducer behavior on optimistic add, success, and failure actions.
- Effect/middleware tests: that the effect dispatches API call and response actions; ordering and side-effects.
- Integration: Store + API mocked to simulate delays, success and failure; UI layer asserting immediate update and final reconciliation/rollback.
- E2E: simulate real UI click with controlled network latency (Cypress/Playwright intercept) to observe optimistic UI and background reconciliation.
- Negative/edge: duplicate submissions, network flakiness, API slow responses, idempotency collisions.
- Security: ensure only authorized calls are made; unauthorized API failure handled gracefully.
- Accessibility: focus and keyboard behavior for submit controls (disable state while pending).

---

## Structured Test Cases

ID: OPT-CTC-0006-01  
Title: Optimistic UI update on Add Custom Item submit (happy path)  
Related Requirement: Component Req - 0006 / User Req - 0006  
Priority: High  
Test Type: Integration / E2E

Preconditions:
- Project page with BOM visible.
- User logged in with edit permission.
- Network is responsive (simulate normal latency).

Test Data:
- Manual item payload: { tempId: "tmp-1", name: "Wood Glue", quantity: 1, unitPrice: 5.00, isManual: true }

Steps:
1. In UI, open "Add Custom Item" form, fill fields as above, press Save.
2. Immediately observe bomList$-backed UI list.

Expected Result (immediate):
- New item appears in BOM list with temporary marker (e.g., greyed or "Saving..." badge).
- Grand Total reflects provisional addition (e.g., +$5.00) with visual hint "provisional" if product requires.
- postCustomItem() API call has been invoked with payload matching the saved fields (tempId or idempotency-key included).

Follow-up (after API success):
3. Mock API to return 201 with persisted item { id: "srv-123", name: "Wood Glue", quantity:1, unitPrice:5.00, created_by: userX, created_at: ... }.
4. Effect receives success, reducer reconciles.

Expected Result (after success):
- Temporary item is replaced/updated with server item (tempId replaced by id "srv-123").
- "Saving..." badge removed; item displays persistent metadata.
- Grand Total remains correct and now shown as confirmed.
- No duplicate items exist.

Postconditions:
- Remove test item.

---

ID: OPT-CTC-0006-02  
Title: Rollback on API failure — optimistic update removed and user notified (revert policy)  
Related Requirement: Failure handling and user notification  
Priority: High  
Test Type: Integration / E2E

Preconditions:
- Same as OPT-CTC-0006-01.
- Rollback policy = revert optimistic item on failure.

Test Data:
- tempItem as above.

Steps:
1. Submit Add Custom Item form; observe optimistic appearance.
2. Mock API to fail with HTTP 500 after delay.
3. Observe effect dispatching AddManualItemFailure action and reducer behavior.

Expected Result:
- The optimistic entry is removed from bomList$ (or marked removed) per revert policy.
- Grand Total reverts to pre-submit value.
- UI displays a clear error notification/toast: "Failed to save 'Wood Glue'. Please retry."
- API was called exactly once.
- No stale provisional entry remains.

Postconditions:
- None.

---

ID: OPT-CTC-0006-03  
Title: Mark failed optimistic item, allow user to retry (mark-failed policy)  
Related Requirement: Alternate rollback policy supporting retries  
Priority: Medium  
Test Type: Integration / E2E

Preconditions:
- Rollback policy = mark failed (do not remove, mark isFailed=true).

Steps:
1. Submit manual item; optimistic item appears.
2. Mock API to return network error.
3. UI shows the item with error badge and action buttons: Retry / Remove.

Expected Result:
- Item remains in list with isFailed flag and visible error UI (e.g., red exclamation).
- Grand Total either excludes or includes the provisional item per product decision; tests assert chosen behavior.
- On clicking Retry, API is invoked again with same tempId/idempotency-key.
- On retry success, item reconciles with server id (as in OPT-CTC-0006-01).

Postconditions:
- Clean up.

---

ID: OPT-CTC-0006-04  
Title: Reducer unit test — optimistic add action updates state immutably  
Related Requirement: Reducer correctness / state immutability  
Priority: High  
Test Type: Unit

Preconditions:
- Reducer initial state with bomList = [ ...existing items ].

Steps:
1. Dispatch AddManualItemOptimistic action with tempItem to reducer.
2. Assert newState.bomList contains tempItem at expected position.
3. Assert original state is not mutated (referential equality checks for unchanged parts).

Expected Result:
- Reducer returns new state with tempItem inserted.
- Tests verify immutability using object equality or deep clone checks.

Postconditions:
- None.

---

ID: OPT-CTC-0006-05  
Title: Effect unit test — effect calls API and dispatches success/failure appropriately  
Related Requirement: Effect/middleware behavior  
Priority: High  
Test Type: Unit

Preconditions:
- Mocked API service with postCustomItem spy/observable.
- Effect wired to listen for AddManualItemOptimistic action.

Steps:
1. Emit AddManualItemOptimistic action in test effect stream.
2. Mock API to return success observable.
3. Verify effect invoked postCustomItem with correct payload and dispatched AddManualItemSuccess with server payload.
4. Repeat with mocked API error and verify AddManualItemFailure dispatched.

Expected Result:
- API called with expected payload.
- Correct actions dispatched on both success and failure paths.

Postconditions:
- None.

---

ID: OPT-CTC-0006-06  
Title: Idempotency and retry: repeated submit does not create duplicate server entries  
Related Requirement: Prevent duplicate server inserts on retries  
Priority: High  
Test Type: Integration / Unit

Preconditions:
- Client provides idempotency-key or tempId in request header/payload.
- PricingStrategy out-of-scope; focus on API behavior.

Steps:
1. Submit Add Custom Item; optimistic UI appears.
2. Simulate network failure; user clicks Retry which triggers effect and second postCustomItem call with same tempId/idempotency-key.
3. Mock server to respond to first failed request as network error but respond to the second request with success creating a single server record.

Expected Result:
- Only one server record created for the provided idempotency-key.
- After success, client reconciles and shows single item.
- Tests assert backend received multiple calls or that idempotency prevented duplicate creation (depends on system); assert final state has one persisted item.

Postconditions:
- None.

---

ID: OPT-CTC-0006-07  
Title: UI disables submit button while optimistic request pending to avoid accidental duplicates (UX)  
Related Requirement: Prevent double submissions via UI state  
Priority: Medium  
Test Type: UI / E2E

Preconditions:
- Add Custom Item form open.

Steps:
1. Fill form and click Save.
2. Immediately click Save again rapidly.

Expected Result:
- First click triggers optimistic update and API call.
- Submit button is disabled or enters pending state to prevent immediate duplicate clicks.
- If double-click occurs due to race, dedup detection via tempId ensures single server record (see OPT-CTC-0006-06).
- Visual indicator (spinner) shown while pending.

Postconditions:
- None.

---

ID: OPT-CTC-0006-08  
Title: Concurrent optimistic adds: order-preservation and reconciliation with server ids  
Related Requirement: Concurrency and ordering guarantees in UI store  
Priority: Medium  
Test Type: Integration

Preconditions:
- User quickly submits three manual items in succession.

Steps:
1. Submit items A, B, C; each results in temp items tmp-A, tmp-B, tmp-C shown immediately.
2. Mock server to respond with server ids in different order (e.g., srv-2, srv-1, srv-3) after varied delays.
3. Observe reconciliation logic.

Expected Result:
- UI initially shows tmp-A,tmp-B,tmp-C in insert order.
- After server responses, each temp item replaced by correct server item (tmp->srv mapping), maintaining UI order or reordering according to product spec (test asserts chosen behavior).
- No duplicates created; all items reconciled.

Postconditions:
- None.

---

ID: OPT-CTC-0006-09  
Title: Persistence and reload: optimistic item confirmed by server remains after page reload  
Related Requirement: Persistence after reconciliation  
Priority: Medium  
Test Type: Integration / E2E

Preconditions:
- Optimistic add succeeded and reconciled with server id srv-123.

Steps:
1. After reconciliation, reload page or re-fetch BOM from server.
2. Verify persisted item present in server-backed response and UI.

Expected Result:
- Item appears as persisted server-side entry with id srv-123.
- No temporary markers present.

Postconditions:
- None.

---

ID: OPT-CTC-0006-10  
Title: Security: unauthorized user cannot perform optimistic add (API returns 403)  
Related Requirement: Authorization enforcement  
Priority: High  
Test Type: Security / Integration

Preconditions:
- User with read-only permission.

Steps:
1. Attempt optimistic add from UI (client still will show optimistic item).
2. Mock API to return 403 Forbidden.

Expected Result:
- Client may show optimistic item immediately, but on API failure (403), the UI removes the item and shows permission error: "You do not have permission to add items."
- Backend rejects request; no DB changes.
- Tests assert proper error handling and no silent failure.

Postconditions:
- None.

---

## Gherkin / BDD Scenarios

Feature: Optimistic UI updates for manual BOM entries
  In order to make the UI feel responsive
  As a project editor
  I want new manual BOM items to appear instantly in the BOM list while the system saves them in the background

Background:
  Given I am a project editor viewing the BOM

Scenario: Optimistic add shows item immediately and reconciles on success
  Given I add "Wood Glue" quantity 1 price 5.00 and click Save
  When the store receives AddManualItemOptimistic action
  Then the BOM shall display the new item immediately with a "Saving..." marker
  And the client shall call POST /api/projects/{projectId}/bom/manual-items
  When the server responds with persisted item id "srv-123"
  Then the UI shall replace the temporary item with the persisted item and remove the "Saving..." marker

Scenario: API failure reverts optimistic change and notifies user
  Given I add "Wood Glue" and the optimistic item appears
  When the server returns HTTP 500 for the save request
  Then the optimistic item is removed (or marked failed) and the user is shown an error notification

Scenario: Retry after failure restores and persists item
  Given the optimistic add failed and item is marked failed
  When I click Retry for the failed item
  Then the client re-sends the request and on success reconciles the item with server id

Scenario: Prevent duplicate creation on retries using idempotency keys
  Given a request was re-sent due to network error
  When the server receives multiple requests with same idempotency-key
  Then only one persisted item is created and the client reconciles to that single persisted item

---

## Implementation / Test Automation Guidance

- Frameworks:
  - Angular + NgRx: test reducers with Jasmine/Karma or Jest; test effects with @ngrx/effects testing utilities and HttpTestingController; use TestScheduler for marbles to simulate timing.
  - React + Redux: use redux-mock-store, redux-saga-test-plan or redux-observable TestScheduler; test reducers and middleware.
  - E2E: Cypress (with cy.intercept for controlled network responses) or Playwright (page.route) to simulate network latency, success/failure, and to assert UI state before/after responses.
- Key testing practices:
  - Use client-generated tempId (UUID) to correlate optimistic item with server response. Tests assert tempId -> server id mapping.
  - Validate that only the Pricing/Cost recalculation and UI update happen immediately; persistent save occurs in background.
  - Assert that the effect calls API exactly once per intended save (unless retries triggered intentionally).
  - Use deterministic test clocks or marbles to simulate delays and ordering deterministically.
  - For idempotency tests, mock server behavior to watch for idempotency-key header (or tempId in payload) and return same persisted resource when duplicate requests arrive.
  - For accessibility, ensure any temporary markers and status updates are announced to screen readers (use aria-live regions); include accessibility assertions in UI tests.
- Example unit-test pseudo-code (Jest-like):
  it('shows optimistic item immediately and reconciles on success', async () => {
    const tempItem = { tempId: 'tmp-1', name: 'Wood Glue', qty:1, price:5.00 };
    store.dispatch(addManualItemOptimistic(tempItem));
    expect(store.selectSnapshot(bomList)).toContainEqual(expect.objectContaining({ id: 'tmp-1', isManual: true }));
    api.postCustomItem.mockResolvedValue({ id: 'srv-123', name: 'Wood Glue', qty:1, price:5.00 });
    await flushPromises();
    expect(store.selectSnapshot(bomList)).toContainEqual(expect.objectContaining({ id: 'srv-123' }));
  });

---

## Edge Cases & Recommendations
- Choose and document a clear rollback policy: revert vs mark-failed; default recommended: revert + show error, with an optional "keep and mark failed" mode for offline-first UX.
- Use idempotency-key or tempId in request to protect against duplicate server inserts.
- Avoid mutating server canonical data until server confirms; persist only after successful API response for long-term storage.
- Consider optimistic total vs confirmed total: show a small badge or suffix ("provisional") to indicate provisional totals until server confirms.
- Implement exponential backoff retries for transient network errors; tests should cover retry behavior and maximum retry attempts.
- Provide user actions for failed entries: Retry, Edit, Remove.
- For offline-capable apps, queue optimistic changes locally and sync in background once online; include tests for offline->online transition.
- Track telemetry on optimistic updates (time-to-confirmation, failure rate) to monitor UX reliability.
- Ensure concurrency safety if store persists optimistic items to local storage — use consistent tempId generation to avoid collisions.
- Provide audit trail linking optimistic tempId to server id for debugging and reconciliation.

---

## Test Data Summary (quick reference)
- tempItem: { tempId: "tmp-1", name: "Wood Glue", quantity:1, unitPrice:5.00, isManual:true }
- server success response: { id: "srv-123", name:"Wood Glue", quantity:1, unitPrice:5.00, created_by:"userX", created_at:"..." }
- server failure: HTTP 500, body { error: "DB error" }
- idempotency-key: same as tempId in header "Idempotency-Key: tmp-1"

---

What I did: I created a complete, self-contained Markdown test-case document for Component Req - 0006 (Optimistic UI Updates). It includes explicit assumptions, unit/integration/E2E test cases (happy path, failure, idempotency, concurrency), Gherkin scenarios, implementation and automation guidance, and edge-case recommendations.

What I can produce next: I can generate (choose one) runnable unit-test skeletons for your store/effects in your framework (NgRx/Jest, Redux/Redux-Saga, Vuex/Jest), or Cypress/Playwright E2E scripts that simulate optimistic updates and background API reconciliation. Tell me which target framework you prefer and I will produce the code.  
```