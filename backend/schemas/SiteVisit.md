# SiteVisit

A persistent counter for the public lookup page. Collection: `sitevisits`. Source: [backend/models/SiteVisit.js](../../../backend/models/SiteVisit.js).

Currently used only to track visits to the `/lookup` page. The schema is generic (`counterKey`-based) so it could host other counters in the future, but in practice there is exactly one document with `counterKey: 'lookup_site_total'`.

```js
mongoose.Schema({
    counterKey: { type: String, required: true, unique: true, index: true, trim: true },
    totalCount: { type: Number, required: true, min: 0 },
    initializedAt: { type: Date, required: true }
}, { timestamps: true });
```

## Fields

| Field | Type | Required | Notes |
|---|---|:-:|---|
| `_id` | ObjectId | auto | |
| `counterKey` | String | ✓ | Counter identifier. Unique. Currently always `lookup_site_total`. |
| `totalCount` | Number | ✓ | Running count. Mongoose `min: 0` validation. |
| `initializedAt` | Date | ✓ | Set once when the counter is first created. |
| `createdAt` | Date | auto | |
| `updatedAt` | Date | auto | |

## Indexes

- `counterKey` (unique)

## Behavior

`POST /api/lookup/visit` ([routes/lookup.js:44-64](../../../backend/routes/lookup.js#L44-L64)):

1. `findOneAndUpdate({ counterKey: 'lookup_site_total' }, { $inc: { totalCount: 1 } }, { new: true })`.
2. If no document exists yet, create one with `totalCount: 1764` (a seed value carried over from a prior counter system) and `initializedAt: new Date()`.
3. Return `{ totalCount }`.

The seed value `1764` is intentional — it preserves the public-facing visit count from a previously-used third-party counter service ("6developer", per the commit message at `0ec1260 feat: implement local visitor counting instead of fetching 6developer (currently down)`).

## Frontend usage

[LookupApp.vue:30-38](../../../frontend/src/views/lookup/LookupApp.vue#L30-L38) calls `POST /api/lookup/visit` once on `onMounted` and displays the returned count in the bottom-right "Tổng số lượt truy cập" badge. There is no debouncing or session check — every page load increments the counter.

## Endpoints that touch this collection

- [`POST /api/lookup/visit`](../../api/endpoints/lookup.md#post-apilookupvisit) — only.
