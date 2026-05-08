# Public views

The public-facing lookup page is the only unauthenticated view in the app. The "embed mode" feature lets it be iframed into other sites without the surrounding chrome.

## LookupApp

[frontend/src/views/lookup/LookupApp.vue](../../../frontend/src/views/lookup/LookupApp.vue). Mounted at `/lookup`.

### Purpose

End-users (typically students or their parents) search for a single certificate by:

1. **School + student ID** — pick from the school dropdown, enter `ma_sinh_vien`.
2. **Full name + date of birth** — enter `ho_va_ten` and `ngay_sinh` (currently the UI for this mode is **commented out** — see "Hidden features" below).

Results are shown in a PrimeVue `DataTable`.

### Endpoints called

| Method | Path | When |
|---|---|---|
| `POST` | `/api/lookup/filter_schools` | `onMounted`, to populate the school dropdown |
| `POST` | `/api/lookup/visit` | `onMounted`, to increment and read the visitor counter |
| `POST` | `/api/lookup` | On "Tra cứu" button click |

Documented in [api/endpoints/lookup.md](../../api/endpoints/lookup.md).

### Components used

PrimeVue: `Fluid`, `AutoComplete` (school picker), `FloatLabel`, `InputText`, `InputMask`, `Button`, `DataTable`, `Column`. Custom: none — it does not use the `FloatingConfigurator` (only `Login.vue` does).

The school dropdown is wired through the `useSchoolSearch` composable so suggestions filter as the user types ([LookupApp.vue:11](../../../frontend/src/views/lookup/LookupApp.vue#L11)).

### Local state

| Variable | Purpose |
|---|---|
| `schoolsList` | All schools from the backend, normalised to `{ _id, name }` |
| `selectedSchool` | Whichever school the user picked |
| `selectSearchField` | Which of the two lookup modes is active (UI for switching is currently commented out) |
| `ma_sinh_vien`, `ho_ten`, `ngay_sinh` | Input fields |
| `searchable` | Computed; true when at least one valid query pair is filled |
| `pressedSearch` | Toggles whether the results panel is shown |
| `searchResults` | The latest API response |
| `visitorCount` | Latest count from `/api/lookup/visit`, shown in the bottom-right badge |
| `isEmbed` | Computed from `route.query.embed === '1'`; switches to a stripped-down layout for iframe embedding |

### Result columns

The results `DataTable` renders these student fields, all from the [Student schema](../../backend/schemas/Student.md): `truong_id` (resolved to school name client-side), `ma_sinh_vien`, `cccd`, `ho_va_ten`, `ngay_sinh`, `noi_sinh`, `xep_loai`, `so_hieu_chung_chi`, `so_vao_so`, `khoa`, `nam`, `so_quyet_dinh`. The `lop` field is fetched but not displayed.

### Hidden features

The HTML for the **name + DOB** lookup mode (the second of the two query forms) is checked in but commented out at [LookupApp.vue:124-135](../../../frontend/src/views/lookup/LookupApp.vue#L124-L135), along with the "Cách 2:" / "Choice 2:" instructional text. The script logic that backs this mode is still present and functional. Re-enabling the UI is a matter of un-commenting the template block.

The **embed mode** (`/lookup?embed=1`) is fully wired but undocumented in the UI. It removes background gradients, padding, and the wrapper card so the form embeds cleanly in an iframe. Used by external sites that want to host the lookup form.

### Visual design

A blue-themed card layout with floating labels. Bottom-right "Tổng số lượt truy cập" badge for the visitor counter. Bottom-center contact line: `Khi cần hỗ trợ, xin liên hệ theo sđt <support-phone> (<support-contact>)`. The colour palette and spacing are hardcoded in `<style scoped>` rather than relying on the PrimeVue theme — this view does not honour the dark-mode toggle.
