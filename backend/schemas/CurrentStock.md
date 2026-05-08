# CurrentStock

The running balance of blank certificates for a given `(year, tier)` pair. Collection: `currentstocks`. Source: [backend/models/CurrentStock.js](../../../backend/models/CurrentStock.js).

There is exactly one document per `(nam, he)` combination — enforced by a compound unique index. The Stats dashboard treats this as the "header" row that floats above the inventory movements list, showing remaining stock and the active serial range.

```js
mongoose.Schema({
    nam: { type: Number, required: true, index: true },
    he: { type: String, required: true, enum: ['Đại học', 'Cao đẳng'], index: true },
    so_luong: { type: Number, required: true, default: 0 },
    so_hieu_tu: { type: String, trim: true, default: '' },
    so_hieu_den: { type: String, trim: true, default: '' }
}, { timestamps: true });

currentStockSchema.index({ nam: 1, he: 1 }, { unique: true });
```

## Fields

| Field | Type | Required | Default | Notes |
|---|---|:-:|---|---|
| `_id` | ObjectId | auto | | |
| `nam` | Number | ✓ | | Year. |
| `he` | String | ✓ | | Education tier. Enum: `Đại học`, `Cao đẳng`. |
| `so_luong` | Number | ✓ | `0` | Current stock count. |
| `so_hieu_tu` | String | — | `''` | Serial range start (current available block). |
| `so_hieu_den` | String | — | `''` | Serial range end (current available block). |
| `createdAt` | Date | auto | | |
| `updatedAt` | Date | auto | | |

## Indexes

- `nam` (single)
- `he` (single)
- **`(nam, he)` compound, unique** — enforces exactly one stock row per year+tier combination.

## Relationships

None enforced. The `(nam, he)` pair is the logical key shared with `Inventory` and `Action`.

## Year lifecycle

The `(nam, he)` uniqueness constraint is what makes `POST /api/admin/create-year` idempotent on its second call: when a new year is requested, the server creates *two* `CurrentStock` documents (one for each `he` value) with `so_luong: 0` ([admin.js:899-903](../../../backend/routes/admin.js#L899-L903)). These two seed rows are also what makes the year "exist" for the purposes of `GET /available-years`, since that endpoint unions the `nam` distincts from `Inventory`, `Action`, and `CurrentStock`.

`DELETE /api/admin/delete-year/:year` removes all rows in all three collections for that year.

## Endpoints that touch this collection

- [`GET /api/admin/inventory/:year/:he`](../../api/endpoints/inventory.md#get-apiadmininventoryyearhe) — read alongside the `Inventory` movements
- [`PUT /api/admin/current-stock/:year/:he`](../../api/endpoints/inventory.md#put-apiadmincurrent-stockyearhe) — upsert (uses `findOneAndUpdate` with `upsert: true`)
- [`POST /api/admin/current-stock`](../../api/endpoints/inventory.md#post-apiadmincurrent-stock) — explicit create (will fail with duplicate-key if the row already exists)
- [`POST /api/admin/create-year`](../../api/endpoints/inventory.md#post-apiadmincreate-year) — seeds the two rows for a new year
- [`DELETE /api/admin/delete-year/:year`](../../api/endpoints/inventory.md#delete-apiadmindelete-yearyear) — deletes all rows for the year
- [`GET /api/admin/available-years`](../../api/endpoints/inventory.md#get-apiadminavailable-years) — unions distinct `nam` values
