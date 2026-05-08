# School endpoints

Admin-only CRUD on the `schools` collection plus two aggregation endpoints. Source file: [backend/routes/admin.js](../../../backend/routes/admin.js).

For the underlying schema, see [backend/schemas/School.md](../../backend/schemas/School.md).

> A near-duplicate of `filter_schools` exists at the public route `POST /api/lookup/filter_schools` ([lookup.md](lookup.md#post-apilookupfilter_schools)). The implementations are functionally identical; the only difference is the auth requirement.

---

## POST `/api/admin/insert_school`

Create a single school.

**Auth:** `admin` or `manager`.

**Request body:**

```json
{
    "ten_truong": "Đại học A",
    "he_dao_tao": "Hệ Đại học"
}
```

`he_dao_tao` is one of: `Hệ Đại học`, `Hệ Cao đẳng`, `Hệ Cao đẳng Sư phạm`. The schema does **not** enforce this — the frontend's Add dialog restricts to the dropdown.

**Success — `200 OK`:**

```json
{ "message": "School inserted successfully" }
```

**No duplicate-name check at the server** — sending the same `ten_truong` twice creates two distinct documents. The frontend ([Dashboard_Schools.vue:143-148](../../../frontend/src/views/admin/Dashboard_Schools.vue#L143-L148)) refuses duplicates client-side, but a direct API caller can bypass that.

**Errors:** uncaught DB errors propagate as unhandled.

**Consumed by:** [Dashboard_Schools.vue:160](../../../frontend/src/views/admin/Dashboard_Schools.vue#L160).

---

## POST `/api/admin/edit_school`

Update one school by `_id` using `$set` over the entire body.

**Auth:** `admin` or `manager`.

**Request body:** must include `_id`. All other fields are merged.

```json
{ "_id": "...", "ten_truong": "New name", "he_dao_tao": "Hệ Đại học" }
```

**Success — `200 OK`:**

```json
{ "message": "Successfully updated school with id <_id>" }
```

**Errors:** none explicitly handled. No 404 when `_id` doesn't exist.

**Consumed by:** [Dashboard_Schools.vue:88](../../../frontend/src/views/admin/Dashboard_Schools.vue#L88).

---

## POST `/api/admin/delete_school`

Delete one school by `_id`.

**Auth:** `admin` or `manager`.

**Request body:**

```json
{ "_id": "<school id>" }
```

**Success — `200 OK`:**

```json
{ "message": "Successfully deleted school with id <_id>" }
```

**No cascade.** Students with `truong_id` matching the deleted school are **not** removed. The frontend's school-delete flow ([Dashboard_Schools.vue:109-131](../../../frontend/src/views/admin/Dashboard_Schools.vue#L109-L131)) calls `delete_students { truong_id }` first, then optionally calls this endpoint based on a checkbox.

**Errors:** none explicitly handled.

**Consumed by:** [Dashboard_Schools.vue:122](../../../frontend/src/views/admin/Dashboard_Schools.vue#L122).

---

## POST `/api/admin/filter_schools`

Search schools by name (case-insensitive substring).

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{ "ten_truong": "string" }
```

Sending `{ "ten_truong": "" }` returns all.

**Success — `200 OK`:**

```json
{
    "data": [
        { "_id": "...", "ten_truong": "...", "he_dao_tao": "Hệ Đại học" }
    ]
}
```

No explicit sort; the frontend sorts client-side with Vietnamese collation.

**Errors:** `500 { "message": "<err>" }` on DB error.

**Consumed by:**
- [Dashboard_Students.vue:16](../../../frontend/src/views/admin/Dashboard_Students.vue#L16) — populate school dropdown
- [Dashboard_Report.vue:47](../../../frontend/src/views/admin/Dashboard_Report.vue#L47) — for resolving `truong_id` → name in chart labels

---

## POST `/api/admin/schools_with_counts`

Return all schools (or filtered by `ten_truong`) with an attached `student_count`.

**Auth:** `admin`, `manager`, or `viewer`.

**Request body:**

```json
{ "ten_truong": "string (optional, case-insensitive substring)" }
```

If `ten_truong` is omitted, all schools are returned.

**Success — `200 OK`:**

```json
{
    "data": [
        {
            "_id": "...",
            "ten_truong": "Đại học A",
            "he_dao_tao": "Hệ Đại học",
            "student_count": 120
        }
    ]
}
```

Sorted alphabetically by `ten_truong` with Vietnamese collation (`localeCompare('vi')`).

**Implementation note:** the count is computed by aggregating `students` grouped by `truong_id`, then mapping over the filtered school list. The aggregation runs over **all** students regardless of the school filter; for large datasets this is more expensive than necessary.

**Errors:** `500 { "message": "<err>" }` on DB error.

**Consumed by:** [Dashboard_Schools.vue:30](../../../frontend/src/views/admin/Dashboard_Schools.vue#L30).

---

## GET `/api/admin/schools_with_courses`

Build a three-level tree of `course → he_dao_tao → schools` with student counts at every level. Powers the Courses dashboard.

**Auth:** `admin`, `manager`, or `viewer`.

**Request:** no body. (It's a `GET`.)

**Success — `200 OK`:**

```json
{
    "data": {
        "65": {
            "courseName": "65",
            "studentCount": 320,
            "trainingTypes": {
                "Hệ Đại học": {
                    "trainingTypeName": "Hệ Đại học",
                    "studentCount": 200,
                    "schools": [
                        { "school_id": "...", "school_name": "Đại học A", "studentCount": 120 },
                        { "school_id": "...", "school_name": "Đại học B", "studentCount": 80 }
                    ]
                },
                "Hệ Cao đẳng": { ... }
            }
        },
        "QH-2020": { ... }
    }
}
```

The top-level `data` is an **object keyed by course name**, not an array. The frontend converts to an array via `Object.values()` ([Dashboard_Courses.vue:34](../../../frontend/src/views/admin/Dashboard_Courses.vue#L34)) before rendering.

**Errors:** `500 { "error": "Failed to fetch data" }` — note this endpoint uses `error`, not `message`. The original exception is logged to `console.error` server-side and not exposed.

**Consumed by:** [Dashboard_Courses.vue:31](../../../frontend/src/views/admin/Dashboard_Courses.vue#L31).
