# School

A partner training unit (Vietnamese: "đơn vị đào tạo" or "trường"). Collection: `schools`. Source: [backend/models/School.js](../../../backend/models/School.js).

```js
mongoose.Schema({
    ten_truong: String,
    he_dao_tao: String
});
```

## Fields

| Field | Type | Required | Notes |
|---|---|:-:|---|
| `_id` | ObjectId | auto | Referenced by `Student.truong_id`. |
| `ten_truong` | String | — | School/unit name. Searched with case-insensitive regex (`{ $regex, $options: 'i' }`) by every filter endpoint. **Not unique** at the schema level — the frontend (`Dashboard_Schools.vue`) does case-insensitive duplicate-name checks before adding/editing, but those checks live entirely client-side. |
| `he_dao_tao` | String | — | Education tier. The Add/Edit dialog restricts to three values: `Hệ Đại học`, `Hệ Cao đẳng`, `Hệ Cao đẳng Sư phạm` ([Dashboard_Schools.vue:134-138](../../../frontend/src/views/admin/Dashboard_Schools.vue#L134-L138)). The schema does not enforce this — direct DB writes can store anything. |

## Indexes

None defined.

## Relationships

- `Student.truong_id → School._id`. App-level only; no `ref` on either side. Cascade-delete is not automatic.

## Aggregation usage

Every endpoint that joins students to schools uses an explicit `$lookup` with `from: 'schools'` (the auto-pluralized collection name from the model). Examples:

- [`POST /api/admin/schools_with_counts`](../../api/endpoints/schools.md#post-apiadminschools_with_counts) — joins students grouped by `truong_id`, attaches student counts to each school document.
- [`GET /api/admin/schools_with_courses`](../../api/endpoints/schools.md#get-apiadminschools_with_courses) — produces a `course → he_dao_tao → schools` tree.
- [`POST /api/admin/export_stats`](../../api/endpoints/students.md#post-apiadminexport_stats) — splits counts by `school.he_dao_tao` for the three certificate-tier columns.
- [`POST /api/admin/export_stats_all`](../../api/endpoints/students.md#post-apiadminexport_stats_all) — joins for school name and tier per (year, course, school).

## Endpoints that touch this collection

- [`POST /api/lookup/filter_schools`](../../api/endpoints/lookup.md) — public read
- [`POST /api/admin/filter_schools`](../../api/endpoints/schools.md#post-apiadminfilter_schools) — admin read
- [`POST /api/admin/schools_with_counts`](../../api/endpoints/schools.md#post-apiadminschools_with_counts)
- [`GET /api/admin/schools_with_courses`](../../api/endpoints/schools.md#get-apiadminschools_with_courses)
- [`POST /api/admin/insert_school`](../../api/endpoints/schools.md#post-apiadmininsert_school)
- [`POST /api/admin/edit_school`](../../api/endpoints/schools.md#post-apiadminedit_school)
- [`POST /api/admin/delete_school`](../../api/endpoints/schools.md#post-apiadmindelete_school)
- [`POST /api/admin/upload`](../../api/endpoints/students.md#post-apiadminupload) — read by `ten_truong` to resolve `_id`
