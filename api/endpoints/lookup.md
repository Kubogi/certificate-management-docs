# Public lookup endpoints

The three endpoints under `/api/lookup/*`, mounted in [server.js:27](../../../backend/server.js#L27). All are **unauthenticated** — they back the public certificate-search page at `/lookup` and are intended for end users.

Source file: [backend/routes/lookup.js](../../../backend/routes/lookup.js).

---

## POST `/api/lookup`

Search for a student certificate. Two query modes are supported:

1. **By school + student ID** — provide `truong_id` and `ma_sinh_vien`.
2. **By name + birth date** — provide `ho_va_ten` and `ngay_sinh`.

You may include both pairs; the server only requires that one of the two pairs is fully present, and then runs `Student.find(input)` with whatever fields you sent.

**Auth:** none (public).

**Request body — mode 1:**

```json
{
    "truong_id": "<School._id, 24-char hex string>",
    "ma_sinh_vien": "string"
}
```

**Request body — mode 2:**

```json
{
    "ho_va_ten": "Nguyễn Văn A",
    "ngay_sinh": "01/01/2000"
}
```

`ho_va_ten` is matched **case-insensitively but exact-match** (anchored regex with regex special chars escaped — see [routes/lookup.js:21-23](../../../backend/routes/lookup.js#L21-L23)). Substring matches and partial names will not work in this endpoint, even though the admin equivalent does substring matching.

`ngay_sinh` must be `dd/mm/yyyy`. The frontend masks input to that format ([LookupApp.vue:130-133](../../../frontend/src/views/lookup/LookupApp.vue#L130-L133), currently commented out).

**Success — `200 OK`:** an **array** of matching `Student` documents. The frontend treats an empty array as "không tìm thấy".

```json
[
    {
        "_id": "...",
        "truong_id": "...",
        "ma_sinh_vien": "20020001",
        "cccd": "...",
        "ho_va_ten": "Nguyễn Văn A",
        "ngay_sinh": "01/01/2000",
        "noi_sinh": "Hà Nội",
        "lop": "QH-2020",
        "xep_loai": "Giỏi",
        "so_hieu_chung_chi": "AAA-001234",
        "so_vao_so": "0001234",
        "khoa": "65",
        "nam": "2024",
        "so_quyet_dinh": "..."
    }
]
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | Neither valid query pair was provided | `{ "message": "Not enough information (School + Student ID or Name + Birthdate)." }` |
| 500 | DB error | `{ "message": "<err>" }` |

**Other behaviors:**

- `truong_id`, if present, is cast to a `mongoose.Types.ObjectId` before query. Invalid hex strings will throw inside the cast and propagate to the unhandled-error path.
- All input fields are passed to `Student.find(input)` — extra fields not in the schema are simply ignored by Mongoose, but extra schema fields *will* further constrain the result.

**Consumed by:** [LookupApp.vue:76](../../../frontend/src/views/lookup/LookupApp.vue#L76).

---

## POST `/api/lookup/filter_schools`

Search the `schools` collection by name. Used to populate the school dropdown on the lookup page.

**Auth:** none (public).

**Request body:**

```json
{ "ten_truong": "string (case-insensitive substring match)" }
```

Sending `{ "ten_truong": "" }` returns all schools.

**Success — `200 OK`:**

```json
{
    "data": [
        { "_id": "...", "ten_truong": "...", "he_dao_tao": "Hệ Đại học" }
    ]
}
```

The order is whatever Mongo returns (no explicit sort). The frontend re-sorts client-side with Vietnamese collation in [LookupApp.vue](../../../frontend/src/views/lookup/LookupApp.vue) via the `useSchoolSearch` composable.

**Errors:** `500 { "message": "<err>" }` on DB error.

**Consumed by:** [LookupApp.vue:15](../../../frontend/src/views/lookup/LookupApp.vue#L15).

> A separate, identical-spec endpoint `POST /api/admin/filter_schools` exists for the authenticated dashboard. The handler bodies are nearly identical; the only difference is the auth requirement.

---

## POST `/api/lookup/visit`

Increment the public visit counter and return the new total.

**Auth:** none (public).

**Request:** no body required.

**Success — `200 OK`:**

```json
{ "totalCount": 1834 }
```

**Behavior:**

- `findOneAndUpdate({ counterKey: 'lookup_site_total' }, { $inc: { totalCount: 1 } }, { new: true })`.
- If no document exists yet (cold start), creates one with `totalCount: 1764` (a seed value carried over from a previously-used third-party counter — see [SiteVisit](../../backend/schemas/SiteVisit.md#behavior)).

**Errors:** `500 { "message": "<err.message or default>" }` on DB error.

**No abuse protection.** The counter increments on every call from any IP, including the same browser refreshing repeatedly.

**Consumed by:** [LookupApp.vue:33](../../../frontend/src/views/lookup/LookupApp.vue#L33).
