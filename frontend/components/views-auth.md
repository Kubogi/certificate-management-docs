# Auth and error views

Three views under `frontend/src/views/pages/`. Only `Login.vue` and `NotFound.vue` are routed; `Error.vue` and `Access.vue` are checked-in template files that aren't reachable.

---

## Login

[frontend/src/views/pages/auth/Login.vue](../../../frontend/src/views/pages/auth/Login.vue). Route: `/auth/login`.

The single sign-in screen for the admin dashboard. Renders without any layout wrapper — it provides its own full-screen background.

**Endpoints called:**
- `POST /api/auth/login`

**Local state:**

| Variable | Purpose |
|---|---|
| `username`, `password` | Form inputs |
| `rememberMe` | Wired into a checkbox in the template, **but the checkbox UI is currently commented out** ([Login.vue:73-78](../../../frontend/src/views/pages/auth/Login.vue#L73-L78)). The handler still writes `localStorage.remember_me` based on its (always-false) value. |
| `loading` | Shows a spinner on the submit button |
| `errorMessage` | Inline red text below the form on failure |

**Login flow** ([Login.vue:17-51](../../../frontend/src/views/pages/auth/Login.vue#L17-L51)):

1. Validate that both fields are non-empty client-side; otherwise show "Please fill in both fields."
2. `POST /api/auth/login`. On success, save `token` and `user` to localStorage.
3. Always write `localStorage.remember_me = 'true'` (the checkbox feature is disabled but the code still runs the truthy branch when ticked — and since the box is commented out, `rememberMe.value` stays `false`, so this actually goes to the else branch and *removes* `remember_me`).
4. `router.push('/admin/dashboard_students')`.
5. On failure, surface `error.response.data.message` (the backend's Vietnamese error string) verbatim.

**Components used:**
- PrimeVue `InputText` for username, `Password` for password (with `:toggleMask="true"` for the show/hide eye icon).
- Custom `<FloatingConfigurator />` ([components/FloatingConfigurator.vue](../../../frontend/src/components/FloatingConfigurator.vue)) — fixed dark-mode + theme-picker buttons in the top-right. This is the only place outside `LookupTopbar` where these controls appear.

**Visual design:** centered card with a blue gradient border. The header reads "Quản lý chứng chỉ" / `<institution>`.

---

## NotFound

[frontend/src/views/pages/NotFound.vue](../../../frontend/src/views/pages/NotFound.vue). Route: `/pages/notfound`.

The standard 404 page. **Note:** the router's catch-all (`to.matched.length === 0`) does not redirect to `/pages/notfound` — it redirects to `/admin/dashboard_students` (if logged in) or `/auth/login` (otherwise). So this route is reachable only by typing the path directly. It functions as a placeholder rather than a true 404.

---

## Unrouted pages (template boilerplate)

Two files exist but are not in the router. They came from the Sakai template and are kept as-is in case someone wants to wire them up later:

- [Error.vue](../../../frontend/src/views/pages/Error.vue) — generic "something went wrong" page
- [Access.vue](../../../frontend/src/views/pages/Access.vue) — "403 access denied" page

The router instead handles auth failures by redirecting to `/auth/login` and 404s via the catch-all redirect. Treat these two files as inactive.
