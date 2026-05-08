# Inventory

A single stock-movement row for blank certificates ("phôi chứng chỉ"). Collection: `inventories`. Source: [backend/models/Inventory.js](../../../backend/models/Inventory.js).

Inventory rows are **scoped by `(nam, he)`** — the Stats dashboard always filters by year and education tier. The `Inventory` collection records *movements* (e.g. "received 500 blank cert forms on this date"); the running balance is held separately in the `CurrentStock` collection.

```js
mongoose.Schema({
    nam: { type: Number, required: true, index: true },
    he: { type: String, required: true, enum: ['Đại học', 'Cao đẳng'], index: true },
    ngay: { type: Date, required: true },
    nghiep_vu: { type: String, required: true, trim: true },
    so_luong: { type: Number, required: true },
    so_hieu_tu: { type: String, required: true, trim: true, default: '' },
    so_hieu_den: { type: String, required: true, trim: true, default: '' }
}, { timestamps: true });
```

## Fields

| Field | Type | Required | Default | Notes |
|---|---|:-:|---|---|
| `_id` | ObjectId | auto | | |
| `nam` | Number | ✓ | | Year. Indexed. |
| `he` | String | ✓ | | Education tier. Enum: `Đại học`, `Cao đẳng`. **Two values only** — `Hệ Cao đẳng Sư phạm` from `School.he_dao_tao` has no equivalent here. Indexed. |
| `ngay` | Date | ✓ | | Date of the movement. Real `Date`, unlike `Student.ngay_sinh`. |
| `nghiep_vu` | String | ✓ | | Free-text label for the operation type ("nhập kho", "xuất kho", etc.). Trimmed. |
| `so_luong` | Number | ✓ | | Quantity moved. |
| `so_hieu_tu` | String | ✓ | `''` | Serial range start. `required: true` is paired with `default: ''`, so the constraint never actually fires — every doc passes. Effectively optional. |
| `so_hieu_den` | String | ✓ | `''` | Serial range end. Same pattern as `so_hieu_tu`. |
| `createdAt` | Date | auto | | From `timestamps: true`. |
| `updatedAt` | Date | auto | | From `timestamps: true`. |

## Indexes

- `nam` (single)
- `he` (single)

No compound `(nam, he)` index, despite every read filtering on both.

## Relationships

None enforced. Both `Inventory` and `CurrentStock` use the `(nam, he)` pair as a logical foreign key into the year/tier dimension, but neither holds a real ref. Year creation/deletion is coordinated through [`POST /api/admin/create-year`](../../api/endpoints/inventory.md#post-apiadmincreate-year) and [`DELETE /api/admin/delete-year/:year`](../../api/endpoints/inventory.md#delete-apiadmindelete-yearyear).

## Behavior gotchas

1. **`filter_inventory` field list does not match the schema.** [`POST /api/admin/filter_inventory`](../../api/endpoints/inventory.md#post-apiadminfilter_inventory) accepts filters on `ma_hang, ten_hang, loai_hang, so_luong, don_gia, truong_id` — none of which exist on this schema. The endpoint will run without error but match nothing meaningful. This is leftover from an older "generic items" data model that was repurposed for certificate stock without updating the filter handler.

2. **`required` + `default: ''`.** `so_hieu_tu` and `so_hieu_den` are listed as required but always default to empty, so a write with neither field still succeeds. Treat them as optional.

3. **Dual write paths.** Inventory rows can be created via `POST /insert_inventory` (legacy bulk, accepts arrays) or `POST /inventory` (single record). Both are authenticated for `admin`/`manager`. The Stats dashboard uses the single-record path.

## Endpoints that touch this collection

- [`POST /api/admin/insert_inventory`](../../api/endpoints/inventory.md#post-apiadmininsert_inventory) — bulk insert
- [`POST /api/admin/edit_inventory`](../../api/endpoints/inventory.md#post-apiadminedit_inventory) — single update by `_id`
- [`POST /api/admin/delete_inventory`](../../api/endpoints/inventory.md#post-apiadmindelete_inventory) — single delete by `_id`
- [`POST /api/admin/filter_inventory`](../../api/endpoints/inventory.md#post-apiadminfilter_inventory) — see "Behavior gotchas" above
- [`GET /api/admin/inventory/:year/:he`](../../api/endpoints/inventory.md#get-apiadmininventoryyearhe) — list by year + tier (used by Stats dashboard)
- [`POST /api/admin/inventory`](../../api/endpoints/inventory.md#post-apiadmininventory) — single create
- [`PUT /api/admin/inventory/:id`](../../api/endpoints/inventory.md#put-apiadmininventoryid) — single update
- [`DELETE /api/admin/inventory/:id`](../../api/endpoints/inventory.md#delete-apiadmininventoryid) — single delete
- [`GET /api/admin/available-years`](../../api/endpoints/inventory.md#get-apiadminavailable-years) — distinct years
- [`POST /api/admin/create-year`](../../api/endpoints/inventory.md#post-apiadmincreate-year) — year existence check
- [`DELETE /api/admin/delete-year/:year`](../../api/endpoints/inventory.md#delete-apiadmindelete-yearyear) — bulk delete by year
