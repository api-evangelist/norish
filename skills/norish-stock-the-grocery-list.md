---
name: norish-stock-the-grocery-list
description: Manage a Norish household's shared grocery list — add items, assign them to a store, tick them off, and undo — including the optimistic-concurrency rules that make it safe.
api: Norish Recipe API
base: https://{norish-instance-host}/api/v1
operations:
- groceryList (GET /api/v1/groceries)
- groceryCreate (POST /api/v1/groceries)
- groceryAssignStore (PATCH /api/v1/groceries/{id}/store)
- groceryMarkDone (PATCH /api/v1/groceries/{id}/done)
- groceryMarkUndone (PATCH /api/v1/groceries/{id}/undone)
- groceryDelete (DELETE /api/v1/groceries/{id})
- storeList (GET /api/v1/stores)
- storeCreate (POST /api/v1/stores)
generated: '2026-08-27'
method: generated
source: https://docs.norish.dev/reference/api
---

# Work the Norish grocery list

The grocery list is shared across a household and syncs in real time to everyone's
devices, including people shopping right now. Two rules follow from that.

## Rule 1 — always send the current `version`

Every grocery mutation that targets a single item — done, undone, store assignment,
delete — requires a `version` field in the request body. It is optimistic concurrency: read
the item, use the `version` you just read, and if someone else changed it in between your
write is the one that must yield. Never cache a `version` across a long-running plan; re-read
`GET /api/v1/groceries` immediately before you write.

## Rule 2 — always send `x-operation-id`

Mutations carrying `x-operation-id` are de-duplicated server-side for 7 days: a replay
returns the stored response instead of adding a second carton of milk. Generate one id per
logical action and reuse it on retries of that same action — never on a new one.

## Steps

1. **Read the list.** `GET /api/v1/groceries`.
2. **Read the stores.** `GET /api/v1/stores`; create one with `POST /api/v1/stores` only if
   the household genuinely lacks it.
3. **Add items.** `POST /api/v1/groceries`, one call per item, each with its own
   `x-operation-id`.
4. **Route items to a store.** `PATCH /api/v1/groceries/{id}/store` with `storeId`, the
   item's `version`, and optionally `savePreference` to remember that this item belongs to
   that store next time.
5. **Tick items off.** `PATCH /api/v1/groceries/{id}/done` with the current `version`.

## Undoing

- A done mark has a named inverse: `PATCH /api/v1/groceries/{id}/undone`. This is the
  cleanest reversal on the whole API — prefer it to deleting and re-adding.
- An added item can be removed with `DELETE /api/v1/groceries/{id}`.
- A store assignment is reversed by assigning a different store, not by an undo call.
- No time window is published for any of these. Do not tell a user they have "24 hours" or
  any other limit; Norish has not stated one.
