# Action

A "transaction" or "action" against the certificate stock — used for issuing, voiding, or otherwise dispositioning blank certificates. Collection: `actions`. Source: [backend/models/Action.js](../../../backend/models/Action.js).

> **Naming gotcha:** the URL paths under `/api/admin/transactions/*` and `/api/admin/*_action` *both* operate on this model. The empty file [models/Transaction.js](../../../backend/models/Transaction.js) does not export anything and is never imported. Always think `Action` when reading these endpoints.

```js
mongoose.Schema({
    nam: { type: Number, required: true, index: true },
    he: { type: String, required: true, enum: ['Đại học', 'Cao đẳng'], index: true },
    noi_dung: { type: String, required: true, trim: true },
    so_luong: { type: Number, required: true },
    so_hieu_tu_den: { type: String, required: true, trim: true }
}, { timestamps: true });
```

## Fields

| Field | Type | Required | Notes |
|---|---|:-:|---|
| `_id` | ObjectId | auto | |
| `nam` | Number | ✓ | Year. Indexed. |
| `he` | String | ✓ | Education tier. Enum: `Đại học`, `Cao đẳng`. Indexed. |
| `noi_dung` | String | ✓ | Description / content. Free-text. Trimmed. |
| `so_luong` | Number | ✓ | Quantity affected. |
| `so_hieu_tu_den` | String | ✓ | Serial range as a single free-text field (e.g. `"00012345 - 00012400"`). Distinct from `Inventory`'s `so_hieu_tu` + `so_hieu_den` pair. Trimmed. |
| `createdAt` | Date | auto | From `timestamps: true`. Used as the natural sort key by `GET /api/admin/transactions/:year/:he`. |
| `updatedAt` | Date | auto | |

## Indexes

- `nam` (single)
- `he` (single)

No compound `(nam, he)` index.

## Behavior gotchas

1. **`filter_actions` field list does not match the schema.** [`POST /api/admin/filter_actions`](../../api/endpoints/transactions.md#post-apiadminfilter_actions) accepts filters on `ma_hanh_dong, loai_hanh_dong, so_tien, ngay_hanh_dong, truong_id` — none of which exist on this schema. The endpoint also sorts by `ngay_giao_dich: -1`, another non-existent field. Like `filter_inventory`, this is leftover from a different domain model. The endpoint runs without error but matches nothing.

2. **Dual write paths.** Actions can be created via `POST /insert_action` (legacy, accepts arrays) or `POST /transactions` (single record). The Stats dashboard uses the latter.

3. **No reference to the related `Inventory` row.** Despite the conceptual relationship between an action ("we issued these certs") and the inventory movement ("we received this block"), there is no foreign key linking them. They share only `(nam, he)`.

## Relationships

None enforced. Logical key: `(nam, he)`.

## Endpoints that touch this collection

- [`POST /api/admin/insert_action`](../../api/endpoints/transactions.md#post-apiadmininsert_action) — bulk insert
- [`POST /api/admin/edit_action`](../../api/endpoints/transactions.md#post-apiadminedit_action) — single update
- [`POST /api/admin/delete_action`](../../api/endpoints/transactions.md#post-apiadmindelete_action) — single delete
- [`POST /api/admin/filter_actions`](../../api/endpoints/transactions.md#post-apiadminfilter_actions) — see "Behavior gotchas"
- [`GET /api/admin/transactions/:year/:he`](../../api/endpoints/transactions.md#get-apiadmintransactionsyearhe) — list by year + tier
- [`POST /api/admin/transactions`](../../api/endpoints/transactions.md#post-apiadmintransactions) — single create
- [`PUT /api/admin/transactions/:id`](../../api/endpoints/transactions.md#put-apiadmintransactionsid) — single update
- [`DELETE /api/admin/transactions/:id`](../../api/endpoints/transactions.md#delete-apiadmintransactionsid) — single delete
- [`GET /api/admin/available-years`](../../api/endpoints/inventory.md#get-apiadminavailable-years) — distinct `nam`
- [`POST /api/admin/create-year`](../../api/endpoints/inventory.md#post-apiadmincreate-year) — existence check
- [`DELETE /api/admin/delete-year/:year`](../../api/endpoints/inventory.md#delete-apiadmindelete-yearyear) — bulk delete
