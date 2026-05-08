# Student

A single certificate record. The collection name is `students`. Source: [backend/models/Student.js](../../../backend/models/Student.js).

The name "student" is slightly misleading — each document is really a certificate issued to a student, and the same person could appear multiple times if they received certificates in multiple years. The unique key is `so_hieu_chung_chi` (certificate serial), not student identity.

## Fields

| Field | Type | Required | Notes |
|---|---|:-:|---|
| `_id` | ObjectId | auto | |
| `truong_id` | ObjectId | ✓ | FK to `schools._id`. Set by the application; no `ref:` defined in the schema, but `$lookup` aggregations in [admin.js](../../../backend/routes/admin.js) explicitly join to the `schools` collection. |
| `ma_sinh_vien` | String | — | Student ID issued by the partner school. Not unique on its own. |
| `cccd` | String | — | Vietnamese citizen ID. |
| `ho_va_ten` | String | — | Full name. |
| `ngay_sinh` | String | — | Date of birth in `dd/mm/yyyy` format. **Stored as a string**, not a `Date`. The Excel import path normalises real `Date` cells to this format ([excel-handler.js:10-20](../../../backend/excel/excel-handler.js#L10-L20)); the UI uses an `InputMask` to enforce the format on entry. |
| `noi_sinh` | String | — | Place of birth. |
| `lop` | String | — | Class designation. |
| `xep_loai` | String | — | Grade/classification ("Giỏi", "Khá", etc.). |
| `so_hieu_chung_chi` | String | ✓ | Certificate serial number. **Unique** at the collection level. |
| `so_vao_so` | String | — | Registry entry number. |
| `khoa` | String | — | Course/cohort ("65", "QH-2020", etc.). Stored as string; year-range exports cast `nam` (not `khoa`) to int. |
| `nam` | String | — | Year of issue. Stored as string but cast to int (`$toInt`) by every aggregation that filters by year range. Plain numeric strings like `"2024"` work; non-numeric values would break range filters silently. |
| `so_quyet_dinh` | String | — | Decision/order number. |

## Indexes

```js
student_schema.index({ so_hieu_chung_chi: 1 }, { unique: true });
```

Only one explicit index. There are no indexes on `truong_id`, `ma_sinh_vien`, `ho_va_ten`, or `nam` despite being filterable — large collections will fall back to collection scans. `filter_students` does sort with a Vietnamese collation (`{ locale: 'vi' }`) which prevents index usage on `khoa` regardless.

## Relationships

- `truong_id → School._id`. Enforced application-side; deleting a school does **not** cascade. The "delete school" button in the UI follows up with a `delete_students { truong_id }` call ([Dashboard_Schools.vue:111](../../../frontend/src/views/admin/Dashboard_Schools.vue#L111)) to clean up.

## Endpoints that touch this collection

- [`POST /api/lookup`](../../api/endpoints/lookup.md) — public read
- [`POST /api/admin/insert_students`](../../api/endpoints/students.md#post-apiadmininsert_students) — bulk insert (single or array)
- [`POST /api/admin/edit_student`](../../api/endpoints/students.md#post-apiadminedit_student) — single update
- [`POST /api/admin/delete_students`](../../api/endpoints/students.md#post-apiadmindelete_students) — bulk delete by filter
- [`POST /api/admin/filter_students`](../../api/endpoints/students.md#post-apiadminfilter_students) — paginated read
- [`POST /api/admin/count_by`](../../api/endpoints/students.md#post-apiadmincount_by) — aggregate by field
- [`POST /api/admin/upload`](../../api/endpoints/students.md#post-apiadminupload) — bulk insert from Excel
- [`POST /api/admin/export_stats`](../../api/endpoints/students.md#post-apiadminexport_stats) — aggregation export
- [`POST /api/admin/export_stats_all`](../../api/endpoints/students.md#post-apiadminexport_stats_all) — detailed aggregation export
- [`POST /api/admin/schools_with_counts`](../../api/endpoints/schools.md#post-apiadminschools_with_counts) — counts by school
- [`GET /api/admin/schools_with_courses`](../../api/endpoints/schools.md#get-apiadminschools_with_courses) — tree by course/system/school
