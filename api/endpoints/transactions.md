# Transaction (Action) endpoints

Admin endpoints for managing certificate-stock actions. The URL paths use the word "transactions" but the underlying Mongoose model is `Action`. The empty file [models/Transaction.js](../../../backend/models/Transaction.js) is dead code — see [backend/README.md](../../backend/README.md#transaction-model).

Source file: [backend/routes/admin.js](../../../backend/routes/admin.js). Schema: [backend/schemas/Action.md](../../backend/schemas/Action.md).

## Two parallel APIs

Like inventory, this domain has two coexisting endpoint families:

1. **Legacy `*_action`** style — bulk insert + filter that does not match the current schema.
2. **REST `transactions/*`** style — single-record CRUD scoped by year and tier. The Stats dashboard uses this family.

Both are wired up.

---

## POST `/api/admin/insert_action`

Bulk-insert action rows.

**Auth:** `admin` or `manager`.

**Request body:** array of `Action` documents.

**Success — `200 OK`:**

```json
{ "message": "Actions inserted successfully" }
```

Duplicate-key errors swallowed (none expected — no unique index).

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** none of the active frontend code.

---

## POST `/api/admin/edit_action`

Update one action by `_id`.

**Auth:** `admin` or `manager`.

**Request body:** must include `_id`; merged via `$set`.

**Success — `200 OK`:**

```json
{ "message": "Successfully updated action with id <_id>" }
```

**Consumed by:** none of the active frontend code.

---

## POST `/api/admin/delete_action`

Delete one action by `_id`.

**Auth:** `admin` or `manager`.

**Request body:**

```json
{ "_id": "..." }
```

**Success — `200 OK`:**

```json
{ "message": "Successfully deleted action with id <_id>" }
```

**Consumed by:** none of the active frontend code.

---

## POST `/api/admin/filter_actions`

**⚠ This endpoint is broken / vestigial.** Like [`filter_inventory`](inventory.md#post-apiadminfilter_inventory), the allow-listed filter fields don't exist on the current `Action` schema.

**Auth:** `admin`, `manager`, or `viewer`.

**Allowed filter fields** ([admin.js:715](../../../backend/routes/admin.js#L715)): `ma_hanh_dong`, `loai_hanh_dong`, `so_tien`, `ngay_hanh_dong`, `truong_id`. None of these are on the `Action` model.

The handler also sorts by `ngay_giao_dich: -1` — also not on the model.

**Effective behavior:** matches everything (since no filters apply), returns the full collection sorted by no-op key. Pagination still works.

**No frontend consumer.**

---

## GET `/api/admin/transactions/:year/:he`

List action rows for a `(year, tier)` pair, sorted by `createdAt` ascending.

**Auth:** `admin`, `manager`, or `viewer`.

**URL params:**
- `:year` — integer.
- `:he` — `Đại học` or `Cao đẳng`. URL-encoded by the frontend.

**Success — `200 OK`:**

```json
{
    "transactions": [
        {
            "_id": "...",
            "nam": 2024,
            "he": "Đại học",
            "noi_dung": "Cấp chứng chỉ cho Đại học A",
            "so_luong": 120,
            "so_hieu_tu_den": "AA-001 - AA-120",
            "createdAt": "...",
            "updatedAt": "..."
        }
    ]
}
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `:he` invalid | `{ "message": "Invalid he parameter. Must be \"Đại học\" or \"Cao đẳng\"" }` |
| 500 | DB error | `{ "message": "<err.message>" }` |

**Consumed by:** [Dashboard_Stats.vue:165](../../../frontend/src/views/admin/Dashboard_Stats.vue#L165).

---

## POST `/api/admin/transactions`

Create a single action.

**Auth:** `admin` or `manager`.

**Request body:** an `Action` document — `nam`, `he`, `noi_dung`, `so_luong`, `so_hieu_tu_den` are all required by the schema.

**Success — `200 OK`:**

```json
{ "message": "Action created successfully", "data": <saved doc> }
```

**Errors:** `500 { "message": "<err.message>" }` — including Mongoose validation errors.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue) (action dialog).

---

## PUT `/api/admin/transactions/:id`

Update a single action by `_id`. Uses `findByIdAndUpdate(id, req.body, { new: true })`.

**Auth:** `admin` or `manager`.

**Success — `200 OK`:**

```json
{ "message": "Action updated successfully", "data": <updated doc> }
```

If `:id` doesn't exist, `data` is `null` and the response is still 200.

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue).

---

## DELETE `/api/admin/transactions/:id`

Delete a single action.

**Auth:** `admin` or `manager`.

**Success — `200 OK`:**

```json
{ "message": "Action deleted successfully" }
```

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue).
