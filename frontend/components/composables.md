# Composables

Two composables in this codebase. Both are simple — neither implements a real store, but `useLayout` happens to behave like one because its state is module-scoped.

---

## `useSchoolSearch(schoolsList)`

[frontend/src/views/SchoolSearch.js](../../../frontend/src/views/SchoolSearch.js).

> The file lives under `views/` rather than `composables/`. Probably a misorganisation; not worth moving without coordination.

### Purpose

Backs PrimeVue `<AutoComplete>` school dropdowns. Filters a list of `{ _id, name }` school objects by case-insensitive substring as the user types, sorted with Vietnamese collation.

### Signature

```js
import { useSchoolSearch } from '@/views/SchoolSearch';

const schoolsList = ref([]);  // populated from /api/admin/filter_schools or /api/lookup/filter_schools
const { filteredSchools, selectedSchoolFilter, searchSchool } = useSchoolSearch(schoolsList);
```

| Returned | Type | Purpose |
|---|---|---|
| `filteredSchools` | `Ref<Array<{ _id, name }>>` | Bind to `:suggestions` on the AutoComplete |
| `selectedSchoolFilter` | `Ref<{ _id, name } | null>` | Bind to `v-model` |
| `searchSchool` | `(event) => void` | Bind to `@complete="searchSchool($event)"` |

### Per-call state

Every call to `useSchoolSearch` returns **fresh** `filteredSchools` and `selectedSchoolFilter` refs. Pass the same `schoolsList` to multiple calls to host parallel dropdowns.

[Dashboard_Students.vue:32-34](../../../frontend/src/views/admin/Dashboard_Students.vue#L32-L34) calls it three times for separate filter / upload / edit dropdowns:

```js
const { filteredSchools, selectedSchoolFilter, searchSchool } = useSchoolSearch(schoolsList);
const { filteredSchools: filteredSchoolsExcel, selectedSchoolFilter: selectedSchoolFilterExcel, searchSchool: searchSchoolExcel } = useSchoolSearch(schoolsList);
const { filteredSchools: filteredSchoolsEdit, selectedSchoolFilter: selectedSchoolFilterEdit, searchSchool: searchSchoolEdit } = useSchoolSearch(schoolsList);
```

### Behaviour details

- Empty query: returns a sorted clone of the full list.
- Non-empty query: filters by `school.name.toLowerCase().includes(query.toLowerCase())`, then sorts the results.
- Sort: `localeCompare(b.name, 'vi')` — Vietnamese collation handles diacritics correctly.

### Used by

- [LookupApp.vue](../../../frontend/src/views/lookup/LookupApp.vue) — public school dropdown.
- [Dashboard_Students.vue](../../../frontend/src/views/admin/Dashboard_Students.vue) — three independent dropdowns.
- [Dashboard_Report.vue](../../../frontend/src/views/admin/Dashboard_Report.vue) — imports but does not actually call it (the `useSchoolSearch` import at line 5 is unused; the schools list is consumed directly).

---

## `useLayout()`

[frontend/src/layout/composables/layout.js](../../../frontend/src/layout/composables/layout.js).

### Purpose

Holds the global layout/theme state. Despite its composable shape, it is effectively a singleton store — both `layoutConfig` and `layoutState` are declared at module scope, so every caller of `useLayout()` shares the same reactive objects.

### State shape

```js
const layoutConfig = reactive({
    preset: 'Aura',
    primary: 'emerald',
    surface: null,
    darkTheme: false,
    menuMode: 'static'         // 'static' | 'overlay'
});

const layoutState = reactive({
    staticMenuDesktopInactive: false,
    overlayMenuActive: false,
    profileSidebarVisible: false,
    configSidebarVisible: false,
    staticMenuMobileActive: false,
    menuHoverActive: false,
    activeMenuItem: null
});
```

### Returned API

| Returned | Type | Purpose |
|---|---|---|
| `layoutConfig` | `Reactive` | Read or write theme settings directly |
| `layoutState` | `Reactive` | Read or write open-state settings directly |
| `toggleMenu` | `() => void` | Toggles overlay or static-menu state depending on `menuMode` and viewport width (cutoff: 991px) |
| `setActiveMenuItem` | `(item) => void` | Sets `activeMenuItem` to either `item.value` or `item` itself |
| `toggleDarkMode` | `() => void` | Flips `layoutConfig.darkTheme` and toggles `.app-dark` on `<html>`. Uses `document.startViewTransition` if available for the cross-fade. |
| `isSidebarActive` | `ComputedRef<boolean>` | True when overlay or mobile sidebar is showing |
| `isDarkTheme` | `ComputedRef<boolean>` | Mirrors `layoutConfig.darkTheme` |
| `getPrimary` | `ComputedRef<string>` | The current primary palette name |
| `getSurface` | `ComputedRef<string \| null>` | The current surface palette name |

### Used by

- [layout/admin/AppLayout.vue](../../../frontend/src/layout/admin/AppLayout.vue) — sidebar/menu state, outside-click handling.
- `AppTopbar.vue`, `AppSidebar.vue`, `AppMenu.vue`, `AppMenuItem.vue`, `AppConfigurator.vue` (in both `layout/admin/` and the unused flat `layout/`) — wired into the layout state.
- [components/FloatingConfigurator.vue](../../../frontend/src/components/FloatingConfigurator.vue) — dark-mode button.
- [Dashboard_Report.vue](../../../frontend/src/views/admin/Dashboard_Report.vue) — reads `getPrimary`, `getSurface`, `isDarkTheme` to update Chart.js colours when the theme changes.

### Caveat

Because the state is module-scoped, **it is not reset across hot-reloads** in Vite dev mode. After a code change that touches `layout.js`, you may need a full page refresh to see consistent menu/dark-mode behaviour.
