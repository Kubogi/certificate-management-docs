# Inventory endpoints

Admin endpoints for managing blank-certificate stock. Two parallel API styles coexist — see "Two parallel APIs" below. Source file: [backend/routes/admin.js](../../../backend/routes/admin.js).

Schemas:
- [Inventory](../../backend/schemas/Inventory.md) — stock movements
- [CurrentStock](../../backend/schemas/CurrentStock.md) — running balance per `(year, tier)`

## Two parallel APIs

Inventory has two coexisting endpoint families:

1. **Legacy filter/CRUD style** — `POST /insert_inventory`, `POST /edit_inventory`, `POST /delete_inventory`, `POST /filter_inventory`. Bulk operations and a search interface that **does not match the current schema** (see [`filter_inventory`](#post-apiadminfilter_inventory)).
2. **REST style** — `GET /inventory/:year/:he`, `POST /inventory`, `PUT /inventory/:id`, `DELETE /inventory/:id`. Single-record operations scoped by year and tier. The Stats dashboard uses this family exclusively.

Both are still wired up. Treat the legacy filter as effectively dead.

---

## POST `/api/admin/insert_inventory`

Bulk-insert inventory rows. Accepts a single object or an array.

**Auth:** `admin` or `manager`.

**Request body:** array of `Inventory` documents — see [schema](../../backend/schemas/Inventory.md#fields).

**Success — `200 OK`:**

```json
{ "message": "Inventory items inserted successfully" }
```

Duplicate-key errors are swallowed and reported as `(duplicates ignored)`. (No unique compound index exists, so this is unlikely to fire in practice.)

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** none of the active frontend code uses this endpoint. Likely vestigial.

---

## POST `/api/admin/edit_inventory`

Update one inventory row by `_id`.

**Auth:** `admin` or `manager`.

**Request body:** must include `_id`; all other fields merged via `$set`.

**Success — `200 OK`:**

```json
{ "message": "Successfully updated inventory item with id <_id>" }
```

**Consumed by:** none of the active frontend code.

---

## POST `/api/admin/delete_inventory`

Delete one inventory row by `_id`.

**Auth:** `admin` or `manager`.

**Request body:**

```json
{ "_id": "..." }
```

**Success — `200 OK`:**

```json
{ "message": "Successfully deleted inventory item with id <_id>" }
```

**Consumed by:** none of the active frontend code.

---

## POST `/api/admin/filter_inventory`

**⚠ This endpoint is broken / vestigial.** It accepts filter fields that **do not exist** on the current `Inventory` schema.

**Auth:** `admin`, `manager`, or `viewer`.

**Allowed filter fields** ([admin.js:663](../../../backend/routes/admin.js#L663)): `ma_hang`, `ten_hang`, `loai_hang`, `so_luong`, `don_gia`, `truong_id`, plus pagination `first` and `size`.

**Schema fields:** `nam`, `he`, `ngay`, `nghiep_vu`, `so_luong`, `so_hieu_tu`, `so_hieu_den`.

Only `so_luong` overlaps. Sending any other documented filter field will produce zero matches. Sorting is by `ten_hang` (also non-existent), which Mongo treats as no-op.

**Success — `200 OK`:**

```json
{ "data": [...], "total": 0 }
```

(Almost certainly always empty.)

**No frontend consumer.**

This endpoint appears to be a leftover from a different "generic items" data model. Ignore until it's removed or rewritten.

---

## GET `/api/admin/inventory/:year/:he`

List inventory rows and the current-stock summary for a `(year, tier)` pair.

**Auth:** `admin`, `manager`, or `viewer`.

**URL params:**
- `:year` — integer (parsed via `parseInt`).
- `:he` — `Đại học` or `Cao đẳng`. **URL-encoded** by the frontend (`encodeURIComponent`).

**Success — `200 OK`:**

```json
{
    "inventory": [
        {
            "_id": "...",
            "nam": 2024,
            "he": "Đại học",
            "ngay": "2024-01-15T00:00:00Z",
            "nghiep_vu": "Nhập kho",
            "so_luong": 500,
            "so_hieu_tu": "AA-001",
            "so_hieu_den": "AA-500"
        }
    ],
    "currentStock": {
        "_id": "...",
        "nam": 2024,
        "he": "Đại học",
        "so_luong": 200,
        "so_hieu_tu": "AA-301",
        "so_hieu_den": "AA-500"
    }
}
```

`inventory` is sorted by `ngay` ascending. If no inventory rows exist, an empty array is returned. If no `currentStock` row exists, a zero-valued stub is returned (not persisted).

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `:he` is not `Đại học` or `Cao đẳng` | `{ "message": "Invalid he parameter. Must be \"Đại học\" or \"Cao đẳng\"" }` |
| 500 | DB error | `{ "message": "<err.message>" }` |

**Consumed by:** [Dashboard_Stats.vue:145](../../../frontend/src/views/admin/Dashboard_Stats.vue#L145).

---

## POST `/api/admin/inventory`

Create a single inventory row.

**Auth:** `admin` or `manager`.

**Request body:** an `Inventory` document — at minimum `nam`, `he`, `ngay`, `nghiep_vu`, `so_luong`. Mongoose validation enforces these (see [Inventory.md](../../backend/schemas/Inventory.md#fields)).

**Success — `200 OK`:**

```json
{ "message": "Inventory record created successfully", "data": <saved doc> }
```

**Errors:** `500 { "message": "<err.message>" }` — including Mongoose validation errors (e.g. missing `nam`).

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue) (the inventory add/edit dialog).

---

## PUT `/api/admin/inventory/:id`

Update a single inventory row by `_id`. Body fields are passed straight to `findByIdAndUpdate(req.params.id, req.body, { new: true })`.

**Auth:** `admin` or `manager`.

**Success — `200 OK`:**

```json
{ "message": "Inventory record updated successfully", "data": <updated doc> }
```

If `:id` doesn't exist, `findByIdAndUpdate` returns `null` and `data` is `null` — the response is still 200.

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue).

---

## DELETE `/api/admin/inventory/:id`

Delete a single inventory row.

**Auth:** `admin` or `manager`.

**Success — `200 OK`:**

```json
{ "message": "Inventory record deleted successfully" }
```

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue).

---

## PUT `/api/admin/current-stock/:year/:he`

Upsert the running stock balance for a `(year, tier)` pair. Idempotent.

**Auth:** `admin` or `manager`.

**URL params:** same as `GET /inventory/:year/:he` — year as int, tier URL-encoded.

**Request body:** any subset of `CurrentStock` fields (`so_luong`, `so_hieu_tu`, `so_hieu_den`). The handler force-overrides `nam` and `he` from the URL params before saving:

```js
findOneAndUpdate({ nam, he }, { ...req.body, nam, he }, { new: true, upsert: true })
```

**Success — `200 OK`:**

```json
{ "message": "Current stock updated successfully", "data": <doc> }
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `:he` invalid | `{ "message": "Invalid he parameter. Must be \"Đại học\" or \"Cao đẳng\"" }` |
| 500 | DB error | `{ "message": "<err.message>" }` |

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue).

---

## POST `/api/admin/current-stock`

Explicit (non-upsert) create. Will fail with E11000 duplicate-key if the `(nam, he)` pair already exists, since `CurrentStock` has a unique compound index.

**Auth:** `admin` or `manager`.

**Request body:** a full `CurrentStock` document.

**Success — `200 OK`:**

```json
{ "message": "Current stock created successfully", "data": <saved doc> }
```

**Errors:** `500 { "message": "<err.message>" }` — including duplicate-key.

**Consumed by:** likely no active frontend usage; the Stats dashboard uses the upsert PUT instead.

---

## GET `/api/admin/available-years`

List all years that have data in any of `Inventory`, `Action`, or `CurrentStock`.

**Auth:** `admin`, `manager`, or `viewer`.

**Success — `200 OK`:**

```json
{ "years": [2024, 2023, 2022] }
```

Sorted descending. Years that exist only as empty `CurrentStock` placeholders (created via `create-year`) still appear here — that's the design.

**Errors:** `500 { "message": "<err.message>" }`.

**Consumed by:** [Dashboard_Stats.vue:102](../../../frontend/src/views/admin/Dashboard_Stats.vue#L102).

---

## POST `/api/admin/create-year`

Create the seed `CurrentStock` rows for a new year — one per `he` value. Refuses if the year already has any data.

**Auth:** `admin` or `manager`.

**Request body:**

```json
{ "year": 2025 }
```

`year` must be a number between 1000 and 3000 (inclusive bounds; [admin.js:886](../../../backend/routes/admin.js#L886)).

**Success — `200 OK`:**

```json
{
    "message": "Year 2025 created successfully with both Đại học and Cao đẳng systems",
    "year": 2025
}
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `year` missing or out of range | `{ "message": "Invalid year parameter" }` |
| 400 | Year already has data in any of the three collections | `{ "message": "Year <year> already has data" }` |
| 500 | DB error | `{ "message": "<err.message>" }` |

**Side effect:** inserts two `CurrentStock` documents — `(year, "Đại học")` and `(year, "Cao đẳng")`, both with `so_luong: 0`.

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue) (new year dialog).

---

## DELETE `/api/admin/delete-year/:year`

Delete **all** `Inventory`, `Action`, and `CurrentStock` rows for the given year.

**Auth:** `admin` or `manager`.

**URL param:** `:year` parsed as int. Must be between 2000 and 2050 (note the tighter range than `create-year`'s 1000–3000 — see audit).

**Success — `200 OK`:**

```json
{
    "message": "Successfully deleted all data for year 2024",
    "deleted": {
        "inventory": 12,
        "actions": 5,
        "currentStock": 2,
        "total": 19
    }
}
```

If no rows existed, the message is `"Year <year> removed from system (no data was present)"` and counts are zero. **The response is always 200**, even when nothing was deleted — by design.

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `:year` falsy or out of 2000-2050 | `{ "message": "Invalid year parameter" }` |
| 500 | DB error | `{ "message": "<err.message>" }` |

**Consumed by:** [Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue) (delete year dialog).
