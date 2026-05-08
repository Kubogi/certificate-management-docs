# Glossary

Field names and string enum values throughout the codebase are in Vietnamese to match the source domain (Vietnamese educational records). This document is the single source of truth for translations. Other docs use the Vietnamese term verbatim.

## Domain terms

| Vietnamese | English | Notes |
|---|---|---|
| chứng chỉ | certificate | The physical document issued to students after completing the `<defence-education>` course. The whole app exists to track these. |
| sinh viên | student | A person who received a certificate. |
| trường / đơn vị (đào tạo) | school / training unit | The partner institution that sent the student. Some UI labels prefer "đơn vị đào tạo" (training unit) over "trường" (school). The DB collection is `schools`. |
| khóa | course / cohort | A class year or batch (often a number like "65", "QH-2020", etc.). |
| hệ đào tạo | training system / education tier | One of three values — see "he_dao_tao enum" below. |
| nghiệp vụ | operation / transaction type | Free-text label on inventory rows describing the operation (e.g. "nhập kho" / receive, "xuất kho" / dispense). |
| phôi chứng chỉ | blank certificate (stock) | Pre-printed certificate forms held in inventory before being filled in. The Stats dashboard tracks these. |
| số hiệu chứng chỉ | certificate serial number | Unique per certificate; the unique key on the `students` collection. |
| số vào sổ | registry entry number | Internal log number when the cert was registered. |
| số quyết định | decision number | The administrative decision/order number authorising the cert issuance. |
| CCCD | citizen ID number | Vietnamese national ID. |
| xếp loại | grade / classification | E.g. "Giỏi" (good), "Khá" (fair). Free-text. |
| nơi sinh | place of birth | |
| ngày sinh | date of birth | Stored as a `String` in `dd/mm/yyyy` format, not as `Date`. See [Date handling](#date-handling). |
| lớp | class | Class designation (free-text). |
| số lượng | quantity | |
| đơn giá | unit price | Referenced by the legacy `filter_inventory` endpoint but absent from the current `Inventory` schema. |
| lượt truy cập | visit count | Counter on the `/lookup` page. |
| tra cứu | lookup / query | The action of searching for a certificate. |

## Database field reference

### `students` collection

| Field | Vietnamese meaning | Type | Notes |
|---|---|---|---|
| `truong_id` | school ID | ObjectId | FK to `schools._id` (required) |
| `ma_sinh_vien` | student ID | String | Issued by the partner school |
| `cccd` | citizen ID | String | |
| `ho_va_ten` | full name | String | |
| `ngay_sinh` | date of birth | String | Format `dd/mm/yyyy` |
| `noi_sinh` | place of birth | String | |
| `lop` | class | String | |
| `xep_loai` | grade | String | |
| `so_hieu_chung_chi` | certificate serial | String | **Unique** (required) |
| `so_vao_so` | registry number | String | |
| `khoa` | course/cohort | String | |
| `nam` | year of issue | String | Stored as string, occasionally cast to int by aggregations |
| `so_quyet_dinh` | decision number | String | |

### `schools` collection

| Field | Meaning | Type |
|---|---|---|
| `ten_truong` | school/unit name | String |
| `he_dao_tao` | education tier | String — see enum below |

### `inventory`, `currentstocks`, `actions` collections

These three collections share the per-year-and-tier structure used to track blank-certificate stock movements.

| Field | Meaning | Type | Used in |
|---|---|---|---|
| `nam` | year | Number | all three |
| `he` | tier (short form) | String enum: `Đại học`, `Cao đẳng` | all three |
| `ngay` | date of operation | Date | inventory only |
| `nghiep_vu` | operation type | String | inventory only |
| `noi_dung` | description / content | String | actions only |
| `so_luong` | quantity | Number | all three |
| `so_hieu_tu` | serial range start | String | inventory, currentstocks |
| `so_hieu_den` | serial range end | String | inventory, currentstocks |
| `so_hieu_tu_den` | serial range (free-text) | String | actions only |

### `users` collection

| Field | Meaning | Type |
|---|---|---|
| `username` | login name | String |
| `password_hash` | bcrypt hash | String |
| `role` | role | String: `admin`, `manager`, `viewer` |

### `sitevisits` collection

| Field | Meaning | Type |
|---|---|---|
| `counterKey` | counter ID (currently always `lookup_site_total`) | String, unique |
| `totalCount` | running visit count | Number |
| `initializedAt` | seed timestamp | Date |

## Enums

### `he_dao_tao` (school tier — long form)

Used on the `School` model and as a label in Excel exports.

- `Hệ Đại học` — University
- `Hệ Cao đẳng` — College
- `Hệ Cao đẳng Sư phạm` — Pedagogical College

### `he` (inventory tier — short form)

Used on `Inventory`, `CurrentStock`, `Action` models and as a route param.

- `Đại học` — University
- `Cao đẳng` — College

> **Inconsistency:** the long form has three values (including pedagogical college); the short form only has two. The Stats dashboard cannot track stock for `Hệ Cao đẳng Sư phạm` schools separately.

### `role`

- `admin` — full access, including user management and the Stats dashboard
- `manager` — can read/write certificates, schools, inventory; cannot manage users or access Stats
- `viewer` — read-only on certificates, schools, inventory; cannot mutate

See [api/README.md](api/README.md#role-matrix) for the per-endpoint matrix.

## Date handling

Dates are not stored as `Date` types in the `students` collection — `ngay_sinh` is a free-text `String` formatted `dd/mm/yyyy`. The frontend uses PrimeVue `InputMask` to enforce that format on input and `format_date()` helpers in views (e.g. [Dashboard_Students.vue:213-223](../frontend/src/views/admin/Dashboard_Students.vue#L213-L223)) to normalise on save. The Excel import path also normalises via `format_date()` in [excel-handler.js:10-20](../backend/excel/excel-handler.js#L10-L20).

`Inventory.ngay` and `SiteVisit.initializedAt` use real `Date`, by contrast. `Action` rows use the Mongoose `timestamps: true` option for `createdAt`/`updatedAt` only.

## UI labels you'll see

A few prominent Vietnamese UI labels and their meanings:

- **"Tra cứu chứng chỉ"** — Look up certificate (lookup page title)
- **"Đơn vị đào tạo"** — Training unit (school dropdown)
- **"Mã sinh viên"** — Student ID
- **"Họ và tên"** — Full name
- **"Quản lý CSDL"** — Database management (admin sidebar group)
- **"Theo dõi phôi chứng chỉ"** — Track blank-certificate stock (Stats dashboard)
- **"Báo cáo thống kê"** — Statistics report (Report dashboard)
- **"Xuất thống kê"** — Export statistics (Excel download)
- **"Đăng nhập"** — Sign in
- **`<institution>`** — `<institution>` (the system owner)
