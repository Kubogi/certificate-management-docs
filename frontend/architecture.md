# Frontend architecture

The whole app is one Vue 3 SPA. There is no shared state library, no service worker, no SSR. This document covers the four cross-cutting concerns: routing, auth, axios, and state.

## Bootstrap

Source: [frontend/src/main.js](../../frontend/src/main.js).

```js
const app = createApp(App);
app.use(router);
app.use(PrimeVue, { theme: { preset: Aura, options: { darkModeSelector: '.app-dark' } } });
app.use(ToastService);
app.use(ConfirmationService);
app.mount('#app');
```

Order of setup:

1. Imports including a single global stylesheet `@/assets/styles.scss`.
2. Two **axios interceptors** registered before `createApp` (see [Axios setup](#axios-setup)).
3. `App.vue` sets `axios.defaults.headers.common['Authorization']` from localStorage on mount ([App.vue:5](../../frontend/src/App.vue#L5)) — a redundancy with the request interceptor, but harmless.
4. PrimeVue config: Aura preset, dark mode triggered by adding/removing the `.app-dark` class on `<html>`.
5. The two PrimeVue services (`Toast`, `Confirmation`) are both wired up; only `Toast` is used in practice.

## Routing

Source: [frontend/src/router/index.js](../../frontend/src/router/index.js). Uses `createWebHistory()` (HTML5 history mode).

### Route table

| Path | Name | Component | Layout | Meta |
|---|---|---|---|---|
| `/lookup` | `lookup` | `views/lookup/LookupApp.vue` | `layout/lookup/LookupLayout.vue` | `requiresAuth: false` |
| `/admin/dashboard_students` | `admin_dashboard_students` | `views/admin/Dashboard_Students.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true` |
| `/admin/dashboard_schools` | `admin_dashboard_schools` | `views/admin/Dashboard_Schools.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true` |
| `/admin/dashboard_report` | `admin_dashboard_report` | `views/admin/Dashboard_Report.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true` |
| `/admin/dashboard_courses` | `admin_dashboard_courses` | `views/admin/Dashboard_Courses.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true` |
| `/admin/user_options` | `admin_user_options` | `views/admin/User_Options.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true` |
| `/admin/dashboard_users` | `admin_dashboard_users` | `views/admin/Dashboard_Users.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true, admin: true` |
| `/admin/dashboard_stats` | `admin_dashboard_stats` | `views/admin/Dashboard_Stats.vue` | `layout/admin/AppLayout.vue` | `requiresAuth: true, admin: true` |
| `/pages/notfound` | `notfound` | `views/pages/NotFound.vue` | none | none |
| `/auth/login` | `login` | `views/pages/auth/Login.vue` | none | none |

All component imports are lazy (`() => import(...)`), giving per-route code splitting.

### Route guard

A single `router.beforeEach` ([router/index.js:102-132](../../frontend/src/router/index.js#L102-L132)) handles three cases:

- **Route requires auth** (`meta.requiresAuth`): hits `GET /api/auth/validate-token` (or `/admin` variant if `meta.admin`). On failure, clears the local token and redirects to `/auth/login`.
- **Already on `/auth/login` with a token**: re-validates; if valid, redirects to `/admin/dashboard_students`. Prevents logged-in users from seeing the login form.
- **Unmatched route** (catch-all): re-validates the token; redirects logged-in users to `/admin/dashboard_students`, others to `/auth/login`.

`isTokenValid(admin)` ([router/index.js:83-100](../../frontend/src/router/index.js#L83-L100)) makes the validation call directly with the token in `Authorization`, **bypassing the axios interceptor for this one specific request** (and explicitly setting the header in the call). Other authenticated requests rely on the interceptor.

### Layouts

Two real layout shells:

- **`layout/admin/AppLayout.vue`** ([source](../../frontend/src/layout/admin/AppLayout.vue)) — sidebar + topbar with `<router-view />` in the main area. Wraps all `/admin/*` routes.
- **`layout/lookup/LookupLayout.vue`** ([source](../../frontend/src/layout/lookup/LookupLayout.vue)) — minimal topbar + main area. Wraps `/lookup`.

Login, NotFound, and the (unrouted) error pages bypass both layouts and render directly. The login page uses `<FloatingConfigurator />` for the dark-mode toggle since there's no topbar to host it.

See [components/layout.md](components/layout.md) for layout component details.

## Auth

### Login flow

1. User submits `username` + `password` at `/auth/login` ([Login.vue](../../frontend/src/views/pages/auth/Login.vue)).
2. `POST /api/auth/login` returns `{ token, user }`.
3. Token saved to `localStorage.auth_token`, user object JSON-serialized into `localStorage.current_user`.
4. Optional `localStorage.remember_me` flag (currently always written as `'true'` because the rememberMe checkbox UI is commented out).
5. `router.push('/admin/dashboard_students')`.

### Logout flow

There is **no logout endpoint and no explicit logout UI.** The route guard clears the token only on validation failure. Effective logout = browser DevTools → Application → localStorage → clear. This is a known gap.

### Token storage

- `localStorage.auth_token` — the JWT (without `Bearer ` prefix).
- `localStorage.current_user` — JSON of the user document returned by login. Includes `_id`, `username`, `role`, and (because the backend returns the full doc) `password_hash`. Read by views to gate write actions and by [User_Options.vue:21-29](../../frontend/src/views/admin/User_Options.vue#L21-L29) for the displayed username.
- `localStorage.remember_me` — never read anywhere. Write-only.

### Role checks

Two layers:

1. **Server side, via the route guard.** Routes with `meta.admin: true` (`dashboard_users`, `dashboard_stats`) call `/api/auth/validate-token/admin`, which returns 403 for non-admins. The guard treats any failure as "not logged in" and redirects to login.

2. **Client side, via `isPrivileged`.** Within views:

   ```js
   const currentUser = JSON.parse(localStorage.getItem('current_user') || '{}');
   const isPrivileged = ['admin', 'manager'].includes(currentUser.role);
   ```

   Used to disable Add/Edit/Delete buttons for `viewer` role. Examples: [Dashboard_Students.vue:9](../../frontend/src/views/admin/Dashboard_Students.vue#L9), [Dashboard_Schools.vue:8](../../frontend/src/views/admin/Dashboard_Schools.vue#L8), [Dashboard_Courses.vue:10](../../frontend/src/views/admin/Dashboard_Courses.vue#L10).

   This is **purely cosmetic** — the disabled buttons can be re-enabled in DevTools, but the backend's role middleware will reject the resulting requests.

### Sidebar gating

[layout/admin/AppMenu.vue:54-60](../../frontend/src/layout/admin/AppMenu.vue#L54-L60) hides the "Quản lý danh sách người dùng" menu item from non-admins. The "Theo dõi phôi chứng chỉ" item (`/admin/dashboard_stats`, also admin-only) is **not** gated in the sidebar — managers and viewers will see the link, click it, and bounce off the route guard back to login. This is a small bug worth flagging.

## Axios setup

Source: [frontend/src/main.js:14-36](../../frontend/src/main.js#L14-L36). Two interceptors:

### Request interceptor

```js
axios.interceptors.request.use((config) => {
    const token = localStorage.getItem('auth_token');
    if (token) config.headers['Authorization'] = token;
    else delete config.headers['Authorization'];
    return config;
});
```

Attaches the token to every outbound request **without** the `Bearer ` prefix — matching what the backend's auth middleware expects.

### Response interceptor

```js
axios.interceptors.response.use((response) => {
    const newToken = response.headers['authorization'];
    if (newToken) localStorage.setItem("auth_token", newToken.split(" ")[1]);
    return response;
});
```

Reads the (possibly-refreshed) token from the response. The backend returns refreshed tokens **with** the `Bearer ` prefix, so the `.split(" ")[1]` strips it before storage. The asymmetry is documented in [backend/auth.md](../backend/auth.md#token-rotation-sliding-refresh).

If the response has no `authorization` header (most responses), the existing token is left alone.

### Base URL

Not set on the axios default. Each view recomputes `api_base_url` from env at module-level:

```js
const api_base_url = import.meta.env.VITE_DEBUG == 0 ? '' : import.meta.env.VITE_API_BASE_URL;
```

Then concatenates it manually on every call: `axios.post(api_base_url + '/api/admin/filter_students', ...)`.

### File upload exception

PrimeVue's `<FileUpload>` component uses XHR directly, bypassing axios. The single use site ([Dashboard_Students.vue:135-138](../../frontend/src/views/admin/Dashboard_Students.vue#L135-L138)) attaches the auth header manually:

```js
const beforeSend = (event) => {
    event.xhr.setRequestHeader('Authorization', localStorage.getItem('auth_token'));
    event.formData.append("ten_truong", selectedSchoolFilterExcel.value.name);
};
```

## State

There is no Pinia, no Vuex, no global store. Persistent and shared state lives in three places:

### `localStorage`

The de-facto session store.

| Key | Type | Written by | Read by |
|---|---|---|---|
| `auth_token` | string (JWT) | login, axios response interceptor, route guard (clears on 401/403) | axios request interceptor, route guard, FileUpload's beforeSend |
| `current_user` | JSON of `User` | login | views (for `isPrivileged` and username display), `AppMenu` (for admin-menu gating) |
| `remember_me` | string `'true'` | login (always, since the checkbox UI is commented out) | nothing |

### Composables

Vue 3 composables that bundle reactive `ref()`/`reactive()` state with helpers. Two exist:

- **`useSchoolSearch(schoolsList)`** ([SchoolSearch.js](../../frontend/src/views/SchoolSearch.js)) — produces `{ filteredSchools, selectedSchoolFilter, searchSchool }`. Each call returns a fresh state instance, so a single view can host multiple parallel school dropdowns. `Dashboard_Students.vue` uses three of them ([Dashboard_Students.vue:32-34](../../frontend/src/views/admin/Dashboard_Students.vue#L32-L34)) — for the filter, the upload dialog, and the edit/add dialog.

- **`useLayout()`** ([layout/composables/layout.js](../../frontend/src/layout/composables/layout.js)) — wraps two **module-level** reactive objects (`layoutConfig` and `layoutState`). Since they live in module scope, every caller of `useLayout()` shares the same state — this composable behaves like a singleton store. Holds dark mode, menu mode, sidebar visibility, primary color, etc.

See [components/composables.md](components/composables.md) for the per-export contract.

### Component-local state

Everything else is component-local: `ref()`, `reactive()`, `computed()`. Cross-component communication is via props/events, but the only meaningful prop chain is layout → router-view. Most "shared" data (e.g. the schools list in `Dashboard_Students.vue`) is fetched independently per view rather than cached centrally.

## Build output

`npm run build` → `frontend/dist/` containing:

- `index.html`
- `assets/<chunk>.js`, `<chunk>.css` — code-split per route plus a vendor chunk
- Static assets

Served as static files by Nginx on the production VPS, with a `try_files … /index.html;` rule so hard refreshes on `/admin/*` paths still load `index.html` and let the SPA router take over. The `frontend/vercel.json` file is leftover from the Sakai template and is not used in production — see [README.md](README.md#what-to-ignore).
