# Frontend

A Vue 3 SPA built on the [Sakai](https://sakai.primevue.org/) admin template (PrimeVue's official starter). Bundled with Vite, styled with Tailwind + SCSS, and using PrimeVue components throughout. There is no state-management library — see [architecture.md](architecture.md#state).

## Stack

From [frontend/package.json](../../frontend/package.json):

| Layer | Choice | Version |
|---|---|---|
| Framework | Vue | 3.4.34 |
| Router | vue-router | 4.5.1 |
| UI library | PrimeVue + PrimeIcons + tailwindcss-primeui | 4.3.4 |
| Theme | @primeuix/themes (Aura preset) | 1.1.1 |
| HTTP | axios | 1.9.0 |
| Charts | chart.js | 3.3.2 |
| Dates | dayjs | 1.11.13 |
| Build | Vite | 5.4.19 |
| CSS | Tailwind 3.4.17 + Sass 1.88.0 | |
| Lint/format | ESLint 8.57.1 + Prettier 3.5.3 | |

No TypeScript. The `name` in `package.json` is `sakai-vue` (from the template); the project hasn't been renamed.

## Commands

All run from `frontend/`:

```sh
npm install          # first-time setup
npm run dev          # Vite dev server (default :5173)
npm run build        # production build → dist/
npm run preview      # preview the production build locally
npm run lint         # eslint --fix on .vue/.js/.jsx/.cjs/.mjs
```

There are no tests and no `test` script.

## Environment

Two build-time env vars, both prefixed `VITE_` (Vite injects only those into the client bundle):

| Variable | Effect |
|---|---|
| `VITE_DEBUG` | When `0`, axios uses an **empty** base URL (same-origin requests, expecting a deploy-time proxy). Any other value (or unset) makes axios use `VITE_API_BASE_URL`. The flag is misnamed — `VITE_DEBUG=0` is actually the *production*-style configuration. |
| `VITE_API_BASE_URL` | Full backend URL, e.g. `http://localhost:5005`. Only consulted when `VITE_DEBUG != 0`. |

The exact pattern is reproduced in every view that makes HTTP calls:

```js
const api_base_url = import.meta.env.VITE_DEBUG == 0 ? '' : import.meta.env.VITE_API_BASE_URL;
```

These are read into the bundle at build time, so changing values requires a rebuild — there is no runtime config injection.

## Folder layout

```
frontend/src/
├── main.js                              # entry point — see architecture.md
├── App.vue                              # root, just <router-view />
├── router/
│   └── index.js                         # all route definitions + guards
├── views/                               # page-level components
│   ├── lookup/
│   │   └── LookupApp.vue                # public lookup page
│   ├── admin/
│   │   ├── Dashboard_Students.vue       # certificate CRUD
│   │   ├── Dashboard_Schools.vue        # school CRUD
│   │   ├── Dashboard_Users.vue          # user CRUD (admin-only route)
│   │   ├── Dashboard_Report.vue         # charts + Excel export
│   │   ├── Dashboard_Stats.vue          # blank-cert inventory (admin-only route)
│   │   ├── Dashboard_Courses.vue        # tree by course/system/school
│   │   └── User_Options.vue             # change own password
│   ├── pages/
│   │   ├── auth/
│   │   │   └── Login.vue
│   │   ├── NotFound.vue
│   │   ├── Error.vue                    # not routed; available as fallback
│   │   └── Access.vue                   # not routed; available as fallback
│   ├── SchoolSearch.js                  # useSchoolSearch composable (mis-located)
│   └── Dashboard.vue                    # TEMPLATE BOILERPLATE — not routed, ignore
├── layout/
│   ├── composables/
│   │   └── layout.js                    # useLayout: dark mode, sidebar state, theme
│   ├── admin/                           # admin shell (sidebar + topbar)
│   │   ├── AppLayout.vue
│   │   ├── AppTopbar.vue
│   │   ├── AppSidebar.vue
│   │   ├── AppMenu.vue
│   │   ├── AppMenuItem.vue
│   │   └── AppConfigurator.vue
│   ├── lookup/                          # minimal shell for the public page
│   │   ├── LookupLayout.vue
│   │   └── LookupTopbar.vue
│   ├── AppLayout.vue                    # legacy template shell — UNUSED
│   ├── AppTopbar.vue, AppSidebar.vue,
│   │   AppMenu.vue, AppMenuItem.vue,
│   │   AppFooter.vue, AppConfigurator.vue   # all template boilerplate, unused
├── components/
│   ├── FloatingConfigurator.vue         # dark-mode + theme button (used on lookup + login)
│   ├── dashboard/                       # TEMPLATE BOILERPLATE — unused
│   │   ├── BestSellingWidget.vue
│   │   ├── NotificationsWidget.vue
│   │   ├── RecentSalesWidget.vue
│   │   ├── RevenueStreamWidget.vue
│   │   └── StatsWidget.vue
│   └── landing/                         # TEMPLATE BOILERPLATE — unused
│       ├── FeaturesWidget.vue
│       ├── FooterWidget.vue
│       ├── HeroWidget.vue
│       ├── HighlightsWidget.vue
│       ├── PricingWidget.vue
│       └── TopbarWidget.vue
└── assets/
    ├── styles.scss                      # global styles entry point
    ├── tailwind.css
    ├── layout/                          # SCSS modules from the Sakai template
    └── demo/                            # template demo styles
```

## What to ignore

The Sakai template ships with a lot of demo content. The following are **not used** by the running app and should be ignored when navigating the codebase:

- All of [components/dashboard/](../../frontend/src/components/dashboard/) — five sales/notification widgets, only imported by the unused `views/Dashboard.vue`.
- All of [components/landing/](../../frontend/src/components/landing/) — six landing-page widgets, no imports anywhere.
- [views/Dashboard.vue](../../frontend/src/views/Dashboard.vue) — defined but never added to the router.
- The flat `layout/AppLayout.vue` (and siblings) at the top of `layout/` — the *real* shells live in `layout/admin/` and `layout/lookup/`. The flat versions appear to be the original Sakai shell, replaced by the per-area variants when this app forked.
- [views/pages/Error.vue](../../frontend/src/views/pages/Error.vue) and [views/pages/Access.vue](../../frontend/src/views/pages/Access.vue) — neither is routed. The router's catch-all redirects to `/auth/login` or `/admin/dashboard_students` instead.

If you find yourself reading any of these to understand a feature, you've gone too far down. Stop.

## Detailed reading order

If you're new:

1. [architecture.md](architecture.md) — routing, auth, state, axios setup.
2. [components/views-public.md](components/views-public.md) — the simplest view (the public lookup page).
3. [components/views-admin.md](components/views-admin.md) — the seven admin dashboards.
4. [components/views-auth.md](components/views-auth.md) — login and error pages.
5. [components/layout.md](components/layout.md) — the admin shell.
6. [components/composables.md](components/composables.md) — `useSchoolSearch` and `useLayout`.
