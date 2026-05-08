# Excel I/O

Bulk student import and statistics export both run through [backend/excel/excel-handler.js](../../backend/excel/excel-handler.js). Two libraries are involved: `node-xlsx` for parsing uploads, `exceljs` for writing styled exports from a template.

## Bulk student import

### Endpoint

`POST /api/admin/upload` ([routes/admin.js:225-264](../../backend/routes/admin.js#L225-L264)). Roles: `admin`, `manager`.

### Multer config

```js
const upload = multer({ dest: path.join(__dirname, '..', 'excel/uploads') });
router.post('/upload', auth_middleware([...]), upload.single('import'), ...)
```

[admin.js:224-225](../../backend/routes/admin.js#L224-L225). The form field is named `import`. Files land in `backend/excel/uploads/` and are deleted via `fs.unlinkSync` after parsing ([admin.js:236](../../backend/routes/admin.js#L236)).

There is no `fileFilter` and no `limits` — `multer` accepts any file and any size. The frontend enforces `accept=".xls,.xlsx"` and `:maxFileSize="50000000"` ([Dashboard_Students.vue:662](../../frontend/src/views/admin/Dashboard_Students.vue#L662)) but the backend does not.

### Parsing

`get_students(truong_id, filename)` ([excel-handler.js:22-45](../../backend/excel/excel-handler.js#L22-L45)) reads every sheet via `node-xlsx.parse(filename, { cellDates: true })` and walks every row, mapping columns by *position* (not by header text):

| Column index | Field | Notes |
|---|---|---|
| 0 | (ignored, usually a row number) | |
| 1 | `ma_sinh_vien` | required by the validator |
| 2 | `cccd` | |
| 3 | `ho_va_ten` | |
| 4 | `ngay_sinh` | normalised to `dd/mm/yyyy` if a `Date` is parsed |
| 5 | `noi_sinh` | |
| 6 | `lop` | |
| 7 | `xep_loai` | |
| 8 | `so_hieu_chung_chi` | required by the validator |
| 9 | `so_vao_so` | |
| 10 | `khoa` | |
| 11 | `nam` | |
| 12 | `so_quyet_dinh` | |

Each row is validated by `validate(col)`:

```js
col.length >= 13 && typeof col[0] == 'number' && typeof col[12] == 'string'
```

[excel-handler.js:5-7](../../backend/excel/excel-handler.js#L5-L7). Rows that fail validation are silently skipped — this is also how header rows are filtered out (their column 0 is text, not a number).

`truong_id` is read separately from the request: the `ten_truong` form field is matched against the `schools` collection to resolve an `_id`, and that `_id` is set on every parsed row ([admin.js:230-234](../../backend/routes/admin.js#L230-L234)).

### Insertion

`Student.insertMany(students, { ordered: false })`. Duplicate-key errors (E11000 on `so_hieu_chung_chi`) are swallowed and reported back to the client as `duplicatedCount`:

```json
{
    "message": "Students inserted successfully (duplicates ignored)",
    "insertedCount": 42,
    "duplicatedCount": 3
}
```

Any other error returns 500.

### Frontend integration

The upload dialog in [Dashboard_Students.vue:656-668](../../frontend/src/views/admin/Dashboard_Students.vue#L656-L668) uses PrimeVue's `FileUpload` component. The `before-send` handler manually attaches the `Authorization` header (since `FileUpload` uses XHR, not the axios interceptor) and appends `ten_truong` to the form data.

## Statistics export

There are **two** export endpoints, both writing to the same `output.xlsx` file:

| Endpoint | Mode | Source |
|---|---|---|
| `POST /api/admin/export_stats` | summary by `category` (`year`, `school`, or `course`), one row per group, three certificate-tier columns + total | [admin.js:267-505](../../backend/routes/admin.js#L267-L505) |
| `POST /api/admin/export_stats_all` | detailed: one row per `(year, course, school)` triple with three certificate-tier counts + total | [admin.js:507-632](../../backend/routes/admin.js#L507-L632) |

Both accept `{ startYear, endYear }` to filter by year range; `export_stats` also accepts `category`. See [api/endpoints/students.md](../api/endpoints/students.md) for the wire format.

### How the export is built

`export_students(worksheetIndex, insertRowIndex, changeSum, inputRows, rangeNote)` ([excel-handler.js:49-119](../../backend/excel/excel-handler.js#L49-L119)):

1. Loads `backend/excel/template.xlsx`.
2. Removes every worksheet except the one at `worksheetIndex`.
3. If `rangeNote` is provided, writes it into cell `A5` (e.g. `(Từ năm 2020 đến năm 2024)`).
4. Inserts `inputRows` starting at `insertRowIndex` (always 9 in current calls — leaves rows 1–8 for header/title content from the template).
5. Sets a fixed row height of `27.75` and adds thin borders to every cell.
6. Auto-widens column D to fit the longest school/course name (min 16.43, otherwise `len + 4`).
7. If `changeSum` is true, writes a totals row directly after the inserted block, summing the three tier columns and the total column. The columns to sum depend on `worksheetIndex`: indexes `[3-6]` for the per-year/per-school/per-course summaries and `[5-8]` for the detailed `export_stats_all` (worksheet 6).
8. Saves to `backend/excel/output.xlsx`.
9. The caller (`res.download(...)`) streams `output.xlsx` to the client with a content-disposition like `certificate_stats_year_<timestamp>.xlsx`.

### Worksheet indexes

The template has multiple sheets; the export logic picks one by index:

| Worksheet index | Used by |
|---|---|
| 1 | `export_stats` with `category=year` |
| 2 | `export_stats` with `category=school` |
| 3 | `export_stats` with `category=course` |
| 6 | `export_stats_all` |

(There is no UI for worksheets 4 or 5 — the template may contain unused sheets.)

### Concurrency caveat

`output.xlsx` is a **single shared file**. Two simultaneous export requests will race: the file written by the first will be overwritten before `res.download` finishes streaming, and one or both clients may receive a corrupt download. This is acceptable for the current single-tenant deployment but worth knowing.

## Template file

[backend/excel/template.xlsx](../../backend/excel/template.xlsx) is a binary file containing the styled template (multiple worksheets, row 5 reserved for the year-range note, row 9 onwards left empty for data). The template is part of the repo and must not be deleted. There is no programmatic way to regenerate it.
