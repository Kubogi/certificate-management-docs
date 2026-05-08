# Admin views

Seven view components live under `frontend/src/views/admin/`. All are wrapped by the admin shell ([layout/admin/AppLayout.vue](../../../frontend/src/layout/admin/AppLayout.vue)) and require authentication. Two require the `admin` role specifically.

All admin views share the same patterns:

- Inline `api_base_url` resolution from `VITE_DEBUG` / `VITE_API_BASE_URL`.
- Inline `currentUser` + `isPrivileged` (`['admin', 'manager'].includes(role)`) for client-side button gating.
- PrimeVue `DataTable` (or `TreeTable`) as the main display, with `Dialog` modals for add/edit/delete.
- `useToast` from PrimeVue for success/error notifications.
- Vietnamese UI strings throughout.

The bullets below cover the *non-shared* details of each view.

---

## Dashboard_Students — certificate CRUD

[frontend/src/views/admin/Dashboard_Students.vue](../../../frontend/src/views/admin/Dashboard_Students.vue). Route: `/admin/dashboard_students`.

The default landing view after login. Shows a paginated, filterable list of certificates and provides single-record CRUD plus bulk Excel upload.

**Endpoints called:**
- `POST /api/admin/filter_schools` — populate the school dropdown
- `POST /api/admin/filter_students` — paginated, filtered list (the lazy-load handler)
- `POST /api/admin/insert_students` — wrap the form payload in an array, send
- `POST /api/admin/edit_student` — single update by `_id`
- `POST /api/admin/delete_students` — delete by `_id`
- `POST /api/admin/upload` — multipart file upload via PrimeVue `FileUpload`

**Notable patterns:**
- Three independent `useSchoolSearch` instances ([Dashboard_Students.vue:32-34](../../../frontend/src/views/admin/Dashboard_Students.vue#L32-L34)) for the filter, edit dialog, and upload dialog — the composable's per-call state model is essential here.
- Lazy-loaded DataTable (`:lazy="true"`, `@page="onPage"`) — pagination always round-trips to the server.
- Date inputs use PrimeVue `InputMask mask="99/99/9999"` to enforce `dd/mm/yyyy`. The `format_date()` helper at [Dashboard_Students.vue:213-223](../../../frontend/src/views/admin/Dashboard_Students.vue#L213-L223) normalises any `Date` objects back to that string format before sending.
- The Excel upload uses `<FileUpload>` directly (not axios), so the `before-send` handler manually attaches the auth header and the `ten_truong` form field.
- Add/edit/delete/upload buttons are gated by `isPrivileged`. The visible filter and search are available to viewers.

---

## Dashboard_Schools — school CRUD

[frontend/src/views/admin/Dashboard_Schools.vue](../../../frontend/src/views/admin/Dashboard_Schools.vue). Route: `/admin/dashboard_schools`.

Lists schools (training units) with their student counts. Supports add/edit/delete. The "delete" flow is two-stage: first delete the school's students, then optionally delete the school record itself based on a checkbox.

**Endpoints called:**
- `POST /api/admin/schools_with_counts` — list with counts
- `POST /api/admin/insert_school`, `edit_school`, `delete_school` — write
- `POST /api/admin/delete_students` — remove students belonging to the deleted school

**Notable patterns:**
- Client-side duplicate-name detection via `isDuplicateAddSchoolName` and `isDuplicateEditSchoolName` (case-insensitive). The backend does not enforce uniqueness, so this is a best-effort check.
- The `he_dao_tao` dropdown is hardcoded with three options: `Hệ Đại học`, `Hệ Cao đẳng`, `Hệ Cao đẳng Sư phạm` ([Dashboard_Schools.vue:134-138](../../../frontend/src/views/admin/Dashboard_Schools.vue#L134-L138)).

---

## Dashboard_Users — user management (admin-only)

[frontend/src/views/admin/Dashboard_Users.vue](../../../frontend/src/views/admin/Dashboard_Users.vue). Route: `/admin/dashboard_users`. **Requires `admin` role** (`meta: { requiresAuth: true, admin: true }`).

CRUD on user accounts. Routed only for admins; the sidebar conditionally adds this menu item only when `userRole === 'admin'` ([AppMenu.vue:54-60](../../../frontend/src/layout/admin/AppMenu.vue#L54-L60)).

**Endpoints called:**
- `POST /api/auth/list_users`
- `POST /api/auth/register` — for "Add user" (the endpoint is admin-only and is reused here despite being under `/auth`)
- `POST /api/auth/update_user`
- `POST /api/auth/delete_user`

**Notable patterns:**
- Roles are presented as Vietnamese labels (`Quản trị`, `Quản lý`, `Người xem`) but stored as English strings (`admin`, `manager`, `viewer`). The mapping is in `roleLabels` ([Dashboard_Users.vue:25-29](../../../frontend/src/views/admin/Dashboard_Users.vue#L25-L29)).
- The edit dialog's password field hint says *"để trống nếu không đổi"* (leave blank if not changing); the backend honors that semantic.

---

## Dashboard_Report — charts and statistics export

[frontend/src/views/admin/Dashboard_Report.vue](../../../frontend/src/views/admin/Dashboard_Report.vue). Route: `/admin/dashboard_report`.

Three Chart.js bar charts (counts by year, school, course) plus four export buttons that download Excel files.

**Endpoints called:**
- `POST /api/admin/filter_schools` — for resolving `truong_id` → school name on the chart axis
- `POST /api/admin/count_by` — three calls, one per chart (`{ field: "nam" }`, `{ field: "truong_id" }`, `{ field: "khoa" }`)
- `POST /api/admin/export_stats` — for "Xuất thống kê theo năm/đơn vị/khóa" buttons (with `category` set per button)
- `POST /api/admin/export_stats_all` — for "Xuất thống kê tổng hợp"

**Notable patterns:**
- Chart colours read from the active PrimeVue theme via CSS custom properties (`--p-primary-500`). Re-evaluated when the theme/dark-mode changes via a `watch` on `[getPrimary, getSurface, isDarkTheme]` ([Dashboard_Report.vue:227-233](../../../frontend/src/views/admin/Dashboard_Report.vue#L227-L233)).
- Excel download is built around `responseType: 'blob'` — the response is wrapped in a `Blob`, given an object URL, then triggered via a programmatic `<a download>` click.
- The year-range inputs feed into both `export_stats` and `export_stats_all`. If left blank, the frontend uses `min(years)..max(years)` from the year chart's labels — which only includes years with student data, not empty years from `available-years`.

**Quirk:** Charts are sized at `width: 50%;` ([Dashboard_Report.vue:305](../../../frontend/src/views/admin/Dashboard_Report.vue#L305)). On small screens this often produces visual overflow. Worth a UX revisit.

---

## Dashboard_Stats — blank-certificate inventory (admin-only)

[frontend/src/views/admin/Dashboard_Stats.vue](../../../frontend/src/views/admin/Dashboard_Stats.vue). Route: `/admin/dashboard_stats`. **Requires `admin` role.**

The largest view in the app (1100+ lines). Manages the year-and-tier-scoped certificate-stock data: inventory movements, current stock balance, and actions/transactions.

> **Sidebar gating bug:** the menu item for this view ([AppMenu.vue:44-48](../../../frontend/src/layout/admin/AppMenu.vue#L44-L48)) is *not* conditionally rendered. Managers and viewers see "Theo dõi phôi chứng chỉ" in the sidebar; clicking it bounces them off the route guard back to login.

**Endpoints called:**

For inventory:
- `GET /api/admin/inventory/:year/:he` — returns `{ inventory, currentStock }` together
- `POST /api/admin/inventory` — single create
- `PUT /api/admin/inventory/:id` — single update
- `DELETE /api/admin/inventory/:id` — single delete
- `PUT /api/admin/current-stock/:year/:he` — upsert the running balance

For transactions:
- `GET /api/admin/transactions/:year/:he`
- `POST /api/admin/transactions`
- `PUT /api/admin/transactions/:id`
- `DELETE /api/admin/transactions/:id`

For year management:
- `GET /api/admin/available-years`
- `POST /api/admin/create-year`
- `DELETE /api/admin/delete-year/:year`

**Notable patterns:**
- A single year + tier dropdown drives every fetch on the page. `onYearChange` and `onHeChange` ([Dashboard_Stats.vue:175-192](../../../frontend/src/views/admin/Dashboard_Stats.vue#L175-L192)) re-fetch both inventory and transactions in parallel when either changes.
- The `selectedHe` dropdown only offers `Đại học` and `Cao đẳng` (matching the inventory schema enum), not the third `Hệ Cao đẳng Sư phạm` option from the schools' `he_dao_tao` enum.
- All currency/quantity numbers are formatted with `vi-VN` locale (`new Intl.NumberFormat('vi-VN').format(...)` at [Dashboard_Stats.vue:96-98](../../../frontend/src/views/admin/Dashboard_Stats.vue#L96-L98)).
- Per-tier URL encoding: every fetch URL-encodes `selectedHe.value` because `Đại học` contains a space (e.g. ``${api_base_url}/api/admin/inventory/${year}/${encodeURIComponent(he)}``).
- Confirms destructive operations via the `useConfirm` composable from PrimeVue (one of the few places `ConfirmationService` is actually used).

---

## Dashboard_Courses — tree view by course

[frontend/src/views/admin/Dashboard_Courses.vue](../../../frontend/src/views/admin/Dashboard_Courses.vue). Route: `/admin/dashboard_courses`.

A three-level expandable tree: course (`khoa`) → training system (`he_dao_tao`) → school. Leaf nodes show student counts and offer a "delete all students of this school in this course" action.

**Endpoints called:**
- `GET /api/admin/schools_with_courses` — fetch the whole tree in one shot
- `POST /api/admin/delete_students` — `{ truong_id, khoa }` to delete leaves

**Notable patterns:**
- The backend returns an *object* keyed by course name; the view normalises to an array via `Object.values()` ([Dashboard_Courses.vue:34](../../../frontend/src/views/admin/Dashboard_Courses.vue#L34)).
- Courses where `khoa` is missing or empty are bucketed under the literal string `"Chưa rõ"` (≈ "unknown").
- Numeric course names are prefixed with `"Khóa "` for display: `"Khóa 65"` instead of `"65"` ([Dashboard_Courses.vue:121](../../../frontend/src/views/admin/Dashboard_Courses.vue#L121)).
- The delete confirmation dialog spells out the full hierarchy ("đơn vị X / hệ Y / khóa Z") before wiping — the operation can affect a lot of records.

---

## User_Options — change own password

[frontend/src/views/admin/User_Options.vue](../../../frontend/src/views/admin/User_Options.vue). Route: `/admin/user_options`.

The only "user settings" view. Reads the current username from `localStorage.current_user` and lets the user change their password.

**Endpoints called:**
- `POST /api/auth/change-password`

**Notable patterns:**
- Three password inputs (`oldPassword`, `newPassword`, `confirmPassword`) — the new/confirm match check is client-side only ([User_Options.vue:37-40](../../../frontend/src/views/admin/User_Options.vue#L37-L40)). The backend doesn't validate that they match.
- Username is read once on `onMounted` and not re-fetched. If the user's username is changed by an admin in another session, this view will keep showing the old name until logout.
- This is the only admin view that doesn't follow the `currentUser` + `isPrivileged` pattern — every authenticated user can change their own password regardless of role.
