# Student endpoints

Admin-only CRUD on the `students` collection plus bulk Excel upload and statistics export. All endpoints under `/api/admin/*`. Source file: [backend/routes/admin.js](../../../backend/routes/admin.js).

For the underlying schema, see [backend/schemas/Student.md](../../backend/schemas/Student.md).

---

## GET `/api/admin/home`

Returns the decoded JWT user — used as a "you are logged in" handshake.

**Auth:** `admin` or `manager`.

**Success — `200 OK`:**

```json
{ "user": { "id": "<user._id>", "iat": ..., "exp": ... } }
```

**Consumed by:** none of the active frontend code calls this. Likely vestigial.

---

## POST `/api/admin/insert_students`

Bulk-insert student records. Accepts a single object or an array.

**Auth:** `admin` or `manager`.

**Request body:** an array of `Student` documents (or a single one). Each must include `truong_id` (string, will be coerced to ObjectId) and `so_hieu_chung_chi` (the unique key). Other fields per [Student schema](../../backend/schemas/Student.md#fields).

```json
[
    {
        "truong_id": "65f1...",
        "ma_sinh_vien": "20020001",
        "ho_va_ten": "Nguyễn Văn A",
        "ngay_sinh": "01/01/2000",
        "so_hieu_chung_chi": "AAA-001234",
        "khoa": "65",
        "nam": "2024"
    }
]
```

**Success — `200 OK`:**

```json
{ "message": "Students inserted successfully" }
```

If some inserts hit duplicate-key (E11000 on `so_hieu_chung_chi`):

```json
{ "message": "Students inserted successfully (duplicates ignored)" }
```

The other documents in the batch are still inserted (`{ ordered: false }`).

**Errors:** `500 { "message": "<err.message>" }` on any non-duplicate failure.

**Consumed by:** [Dashboard_Students.vue:327](../../../frontend/src/views/admin/Dashboard_Students.vue#L327) (the "Add certificate" dialog wraps the single object in an array before sending).

---

## POST `/api/admin/edit_student`

Update one student by `_id` using `$set` over the entire body.

**Auth:** `admin` or `manager`.

**Request body:** must include `_id`. All other fields are merged via `$set`.

```json
{
    "_id": "...",
    "ho_va_ten": "Updated name",
    "ngay_sinh": "02/02/2000"
}
```

**Success — `200 OK`:**

```json
{ "message": "Successfully updated student with id <_id>" }
```

**Errors:** none explicitly handled — DB exceptions propagate as unhandled. There is no 404 if `_id` doesn't exist; `updateOne` returns `matchedCount: 0` but the handler still responds 200.

**Consumed by:** [Dashboard_Students.vue:240](../../../frontend/src/views/admin/Dashboard_Students.vue#L240).

---

## POST `/api/admin/delete_students`

Delete students matching the body filter (used both for single-row delete by `_id` and bulk delete by `truong_id` and/or `khoa`).

**Auth:** `admin` or `manager`.

**Request body:** any combination of fields that should constrain the delete.

```json
{ "_id": "<single student id>" }
```

```json
{ "truong_id": "<school id>" }
```

```json
{ "truong_id": "<school id>", "khoa": "65" }
```

`truong_id`, if present, is cast to `ObjectId`. Other fields are passed straight to `Student.deleteMany`.

**Success — `200 OK`:**

```json
{ "message": "Students deleted successfully" }
```

**No `deletedCount`** is returned. Sending an empty body would delete **every student**; the frontend never does this, but there is no server-side guard.

**Consumed by:**
- [Dashboard_Students.vue:261](../../../frontend/src/views/admin/Dashboard_Students.vue#L261) — single delete by `_id`
- [Dashboard_Schools.vue:111](../../../frontend/src/views/admin/Dashboard_Schools.vue#L111) — when deleting a school, first delete all its students
- [Dashboard_Courses.vue:72](../../../frontend/src/views/admin/Dashboard_Courses.vue#L72) — delete all students in a `(truong_id, khoa)` pair

---

## POST `/api/admin/filter_students`

Paginated, filtered list of students.

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{
    "first": 0,
    "size": 30,
    "ma_sinh_vien": "string",
    "ho_va_ten": "string",
    "ngay_sinh": "string",
    "cccd": "string",
    "truong_id": "<school id>",
    "so_hieu_chung_chi": "string",
    "so_vao_so": "string",
    "xep_loai": "string",
    "nam": "string",
    "khoa": "string",
    "so_quyet_dinh": "string"
}
```

All filter fields are optional. Defaults: `first = 0`, `size = 30`. Fields not in the allow-list ([admin.js:47](../../../backend/routes/admin.js#L47)) are silently ignored.

**Filter semantics:**

- `truong_id`: cast to `ObjectId` and compared with `$eq`.
- `khoa`: anchored case-insensitive exact match (so `"65"` will match `"65"` but not `"650"`). Regex special characters in the input are escaped.
- All other fields: case-insensitive substring match (`{ $regex: value, $options: 'i' }`). No escaping — a value containing regex metacharacters can change the search behavior.

**Sorting:** ascending by `khoa`, with Vietnamese collation (`{ locale: 'vi' }`).

**Success — `200 OK`:**

```json
{
    "data": [<Student document>, ...],
    "total": 1234
}
```

`total` is the **unpaginated** count for the same filter (used by the DataTable for pagination).

**Errors:** `500 { "message": "<err>" }` on DB error.

**Consumed by:** [Dashboard_Students.vue:92](../../../frontend/src/views/admin/Dashboard_Students.vue#L92).

---

## POST `/api/admin/count_by`

Aggregate students grouped by an arbitrary field (e.g. count by year, by school, by course).

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{ "field": "nam" }
```

Common values: `nam`, `khoa`, `truong_id`. The field name is interpolated directly into the aggregation pipeline (`'$' + req.body.field`); any string is accepted.

**Success — `200 OK`:** the raw aggregation result, sorted ascending by `_id`.

```json
[
    { "_id": "2020", "count": 120 },
    { "_id": "2021", "count": 150 },
    { "_id": "2022", "count": 200 }
]
```

For `field: "truong_id"`, `_id` is an ObjectId (the frontend resolves it to a school name client-side via [schoolsList](../../../frontend/src/views/admin/Dashboard_Report.vue#L80)).

**Errors:** `500 { "error": "<err.message>" }` — note this endpoint uses `error`, not `message`.

**Consumed by:** [Dashboard_Report.vue:64,77,93](../../../frontend/src/views/admin/Dashboard_Report.vue#L64).

---

## POST `/api/admin/upload`

Bulk-import students from an Excel file. `multipart/form-data`.

**Auth:** `admin` or `manager`.

**Request:**

| Field | Type | Notes |
|---|---|---|
| `import` | file | `.xls` or `.xlsx`. Frontend enforces 50 MB; backend has no limit. |
| `ten_truong` | string | School name. Resolved server-side to `_id` via `School.findOne({ ten_truong })`. |

The Excel parsing is documented in [backend/excel.md](../../backend/excel.md#bulk-student-import). Briefly: positional column mapping, every sheet processed, header rows automatically skipped.

**Success — `200 OK`:**

```json
{
    "message": "Students inserted successfully",
    "insertedCount": 42,
    "duplicatedCount": 0
}
```

If any rows hit duplicate-key (E11000 on `so_hieu_chung_chi`):

```json
{
    "message": "Students inserted successfully (duplicates ignored)",
    "insertedCount": 39,
    "duplicatedCount": 3
}
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | No file in the request | `{ "message": "No file uploaded." }` |
| 500 | Non-duplicate insert error | `{ "message": "<err.message>" }` |

**Edge case:** if `ten_truong` doesn't match any school, `School.findOne` returns `null` and the subsequent `.then(results => { _id = results._id })` throws a `TypeError`. The handler does not catch this case and the response will be a 500 with no useful message.

**Consumed by:** [Dashboard_Students.vue:662](../../../frontend/src/views/admin/Dashboard_Students.vue#L662) via PrimeVue `FileUpload`.

---

## POST `/api/admin/export_stats`

Generate an Excel file aggregating certificate counts by **one** of three categories: year, school, or course. Optional year-range filter.

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{
    "category": "year | school | course",
    "startYear": 2020,
    "endYear":   2024
}
```

`startYear` and `endYear` are optional (and integers; the frontend sends `Number()`-coerced strings). When both are present, the aggregation filters students to that range based on `nam` cast to int. When `start === end`, the title becomes `(Năm <start>)` instead of `(Từ năm <start> đến năm <end>)`.

When omitted, the dataset is the full `students` collection.

**Success — `200 OK`:** binary `.xlsx` body, with `Content-Disposition: attachment; filename="certificate_stats_<category>_<timestamp>.xlsx"`.

The output workbook is built from `template.xlsx`, picking worksheet 1/2/3 for year/school/course respectively. Each row is `[idx, group_value, count_DH, count_CD, count_CD_SP, total, '']` — three certificate-tier columns plus total. A summary row is added below the data with column-wise sums.

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `category` not in `year`/`school`/`course` | `{ "message": "Invalid category" }` |
| 400 | `startYear`/`endYear` provided but `NaN` | `{ "message": "Invalid year values" }` |
| 500 | Aggregation or Excel write error | `{ "message": "<err.message>" }` |

**Concurrency note:** the output writes to the shared `backend/excel/output.xlsx`. Concurrent calls can corrupt one another's downloads — see [backend/excel.md](../../backend/excel.md#concurrency-caveat).

**Consumed by:** [Dashboard_Report.vue:249](../../../frontend/src/views/admin/Dashboard_Report.vue#L249) (three buttons: Year/School/Course).

---

## POST `/api/admin/export_stats_all`

Detailed export: one row per `(year, course, school)` triple, with three certificate-tier columns and a total. Always uses the year-range filter — the frontend defaults to `min(years)..max(years)` if not specified ([Dashboard_Report.vue:240-243](../../../frontend/src/views/admin/Dashboard_Report.vue#L240-L243)).

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{ "startYear": 2020, "endYear": 2024 }
```

**Success — `200 OK`:** binary `.xlsx`, `Content-Disposition: attachment; filename="certificate_stats_all_<timestamp>.xlsx"`. Uses worksheet 6 of the template.

Sort order: by `nam` ascending, then by `khoa` (numbers first, then strings, both Vietnamese-collated), then by `ten_truong` (Vietnamese-collated).

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | `startYear`/`endYear` missing or `NaN` | `{ "message": "Invalid year values" }` |
| 500 | Aggregation or Excel write error | `{ "message": "<err.message>" }` |

**Consumed by:** [Dashboard_Report.vue:249](../../../frontend/src/views/admin/Dashboard_Report.vue#L249) (the "Xuất thống kê tổng hợp" button).
