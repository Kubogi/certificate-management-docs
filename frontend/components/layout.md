# Layouts and shared chrome

There are two real layout shells in this app, one for the public lookup and one for the admin dashboard. Both are children of routes in [router/index.js](../../../frontend/src/router/index.js); the actual page views render through `<router-view />` inside the shell.

A separate set of "flat" layout files exists at the top level of [frontend/src/layout/](../../../frontend/src/layout/) (`AppLayout.vue`, `AppTopbar.vue`, `AppSidebar.vue`, etc.). These are the original Sakai template shells; they have been **superseded by the per-area variants** under `layout/admin/` and `layout/lookup/`. Treat them as unused — none of the active routes reference them.

---

## Admin shell — `layout/admin/AppLayout.vue`

[frontend/src/layout/admin/AppLayout.vue](../../../frontend/src/layout/admin/AppLayout.vue).

Wraps every `/admin/*` route. Composed of:

- `<AppTopbar>` — top bar with menu-toggle button, app title, and the dark-mode toggle
- `<AppSidebar>` — left rail containing `<AppMenu>` (the navigation list)
- `<router-view>` inside `.layout-main-container > .layout-main`
- `<Toast />` — global toast container (the backing store is the PrimeVue `ToastService`)

Layout state — sidebar collapsed/expanded/overlay, mobile vs desktop, active menu item — is held in the `useLayout()` composable's module-level `layoutConfig` and `layoutState` objects ([layout/composables/layout.js](../../../frontend/src/layout/composables/layout.js)). See [composables.md](composables.md#uselayout).

A `watch(isSidebarActive, ...)` ([AppLayout.vue:11-17](../../../frontend/src/layout/admin/AppLayout.vue#L11-L17)) attaches a document-level click listener while the mobile/overlay sidebar is open, so a click outside closes it. The listener is removed when the sidebar closes.

> **Bug worth noting:** [AppLayout.vue:43](../../../frontend/src/layout/admin/AppLayout.vue#L43) calls `removeEventListener('click', outsideClickListener)` — passing the *ref*, not its `.value`. The handler is never actually removed; subsequent open/close cycles accumulate listeners. Functionally harmless because the closures all do the same thing, but it leaks memory over a long session.

### `AppTopbar`

Header bar. Contains:

- The hamburger menu button (toggles the sidebar via `useLayout().toggleMenu`).
- The app title ("Quản lý chứng chỉ" or similar — text varies by version).
- A dark-mode toggle button.

### `AppSidebar` + `AppMenu` + `AppMenuItem`

The left rail. `AppSidebar.vue` is a thin wrapper that holds `<AppMenu />`. `AppMenu.vue` ([source](../../../frontend/src/layout/admin/AppMenu.vue)) defines the navigation tree as a static array:

```js
const model = ref([
    {
        label: 'Người dùng',
        items: [
            { label: 'Cài đặt người dùng', icon: 'pi pi-fw pi-user', to: '/admin/user_options' }
            // 'Quản lý danh sách người dùng' added below if userRole === 'admin'
        ]
    },
    {
        label: 'Quản lý CSDL', icon: 'pi pi-fw pi-briefcase', to: '/pages',
        items: [
            { label: 'Quản lý danh sách chứng chỉ', icon: 'pi pi-fw pi-server', to: '/admin/dashboard_students' },
            { label: 'Quản lý đơn vị liên kết',     icon: 'pi pi-fw pi-address-book', to: '/admin/dashboard_schools' },
            { label: 'Quản lý danh sách khóa học',   icon: 'pi pi-fw pi-book', to: '/admin/dashboard_courses' },
            { label: 'Báo cáo thống kê',             icon: 'pi pi-fw pi-chart-bar', to: '/admin/dashboard_report' },
            { label: 'Theo dõi phôi chứng chỉ',      icon: 'pi pi-fw pi-table', to: '/admin/dashboard_stats' }
        ]
    }
]);
```

The user-management menu item is conditionally appended to the "Người dùng" group when `userRole === 'admin'` ([AppMenu.vue:54-60](../../../frontend/src/layout/admin/AppMenu.vue#L54-L60)).

The "Theo dõi phôi chứng chỉ" item (`/admin/dashboard_stats`) is **shown to all roles**, but the route itself is admin-only — see [architecture.md sidebar gating](../architecture.md#sidebar-gating).

`AppMenuItem.vue` is the recursive item renderer (handles nested items, the active class, etc.).

### `AppConfigurator`

[frontend/src/layout/admin/AppConfigurator.vue](../../../frontend/src/layout/admin/AppConfigurator.vue) (and the parallel [layout/AppConfigurator.vue](../../../frontend/src/layout/AppConfigurator.vue) used by `FloatingConfigurator`).

A small panel that lets the user pick the primary colour and surface palette of the PrimeVue theme. Backed by `useLayout()` — changes to `layoutConfig.primary` are reactive across every component that reads from `getPrimary`.

---

## Lookup shell — `layout/lookup/LookupLayout.vue`

[frontend/src/layout/lookup/LookupLayout.vue](../../../frontend/src/layout/lookup/LookupLayout.vue).

Far simpler than the admin shell — it's just `<LookupTopbar />`, the `<router-view />`, and a `<Toast />`. No sidebar, no menu, no role-aware logic.

The topbar ([LookupTopbar.vue](../../../frontend/src/layout/lookup/LookupTopbar.vue)) shows the centre's name and a `FloatingConfigurator` for the dark-mode toggle. Note that the lookup view itself uses hardcoded blue styling ignored by the dark theme — the toggle changes the topbar but not the form.

---

## FloatingConfigurator

[frontend/src/components/FloatingConfigurator.vue](../../../frontend/src/components/FloatingConfigurator.vue).

A standalone fixed-position widget in the top-right corner. Two buttons:

1. **Dark-mode toggle** — calls `useLayout().toggleDarkMode()` which adds/removes `.app-dark` on `<html>` (with View-Transitions API support if available).
2. **Palette button** — opens [layout/AppConfigurator.vue](../../../frontend/src/layout/AppConfigurator.vue) via a PrimeVue `v-styleclass` directive (`enterFromClass: 'hidden'`, etc.).

Used by:

- The login page ([Login.vue:55](../../../frontend/src/views/pages/auth/Login.vue#L55)) — there's no topbar to host these buttons.
- The lookup topbar (transitively).
- The unrouted error/access pages.

It is **not** used inside the admin shell — there, the dark-mode toggle lives in the topbar and the palette picker in `AppConfigurator` is anchored to the topbar's gear icon instead.
