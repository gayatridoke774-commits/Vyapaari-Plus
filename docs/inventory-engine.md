# Vyapaari+ Inventory Engine

## 1. Scope

The inventory module supports stock transfers between branches, transfer delivery documents, carrier tracking, receipt recording, and discrepancy reconciliation.

The repository uses SQLAlchemy ORM and SQLite by default. Its README describes a desktop application with an in-process service layer. The inventory service methods documented here are Python methods, not HTTP endpoints.

## 2. Data model

| Model | Purpose |
|---|---|
| `Branch` | Identifies a source or destination branch. |
| `StockBatch` | Stores product batch quantities and pricing per branch. |
| `StockTransfer` | Tracks the transfer lifecycle, branches, delivery documents, and carrier details. |
| `TransferItem` | Stores dispatched, good-received, and damaged quantities per product. |
| `CarrierLog` | Records dispatch, transit, delivery, and tracking events. |
| `ReconciliationLog` | Stores the transfer variance and reconciliation summary. |
| `ReconciliationApproval` | Preserves approval or rejection decisions and their audit details. |

Database constraints validate transfer quantities and prevent a transfer from using the same source and destination branch. Reconciliation records are unique per transfer.

## 3. Transfer lifecycle

The workflow includes these states:

1. **DRAFT** — create and validate a transfer request.
2. **DISPATCHED** — deduct source stock and generate a delivery challan and gate pass.
3. **IN_TRANSIT** — record that the dispatched transfer has entered transit.
4. **RECEIVED** — validate receipt quantities and add only good units to destination stock.
5. **RECONCILED** — record a transfer with no variance.
6. **DISPUTED** — record a transfer with a variance requiring resolution.

A transfer with a discrepancy may require an approval decision before it can be marked resolved. Consult `reconciliation.py` for the precise approval rules.

## 4. Stock accounting

For each transfer item:

- Good received units increase usable stock at the destination.
- Damaged units are recorded but are not added to usable destination stock.
- Missing units are calculated as dispatched minus good received minus damaged.
- Reconciliation records the difference between dispatched and good received quantities and calculates the associated loss value.

The source stock is deducted during dispatch. Reconciliation is a bookkeeping and audit step; it does not apply a second stock adjustment.

## 5. Transaction and failure handling

Dispatch, receipt, and reconciliation use SQLAlchemy transactions with commit and rollback handling. Invalid transfer states and invalid receipt quantities are rejected.

The transfer service uses `with_for_update()` on selected queries. Its presence alone does not establish effective row-level locking on SQLite. Concurrent stock updates must be tested against the actual database backend used in deployment.

Transfer status checks prevent ordinary repeat dispatch or receipt after the transfer has advanced. They are not a substitute for a complete idempotency-key implementation that returns the original successful result for a retried request.

## 6. Automated tests

Run the test suite from the repository root:

```bash
python -m pytest -q
>>

The inventory tests cover transfer lifecycle behavior, delivery documents, carrier events, invalid quantities and states, stock deduction and receipt, damaged and missing units, reconciliation, and duplicate reconciliation.

The full suite passed 469 tests in the development checkout on 2026-10-21. Re-run it after any code changes; this historical result does not guarantee that future changes pass.

## 7. Current limitations and follow-up work

The following items were not verified as implemented in the current checkout:

- A REST API, OpenAPI specification, or Postman collection for inventory.
- Persistent idempotency keys and replay of prior successful responses.
- Tracked Alembic migration configuration and migration revisions.
- Effective concurrent-write guarantees on a production database backend.
- Dedicated automated concurrency and timeout tests.

The repository README mentions Alembic support, but the inspected tracked files did not include Alembic configuration or migration revisions. Confirm the project owner's intended migration workflow before adding one.

These limitations should be addressed according to the application's intended architecture rather than by claiming unsupported capabilities.
