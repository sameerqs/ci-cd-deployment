# Pet2text — Client

Next.js 16 App Router frontend of the Pet2text SaaS starter. Ships a complete
auth flow, a role-aware dashboard shell, and two living reference modules
(`users`, `settings`) that demonstrate every project pattern.

## Stack

- Next.js 16 (App Router, RSC default) + React 19
- Tailwind CSS v4 + shadcn/ui (Radix primitives, lucide icons)
- TanStack Table v8 (+ react-virtual)
- react-hook-form + zod
- nuqs (URL state)
- next-themes
- jwt-decode (cookie-stored access token)

No client data cache (no React Query / Redux / Zustand) — reads happen in
async RSCs, writes in `'use server'` actions with `revalidatePath()`.

## Project rules

- **Numeric enums everywhere.** `lib/enum.ts` mirrors the backend's
  `src/common/constants/*` exactly (UserRole: SuperAdmin=0, Admin=1, User=2).
- **Global response envelope.** Every backend endpoint returns
  `{ isSuccess, message, data, errors, warningHeading, warningMessage,
  warningType }`. The `call()` helper in `lib/utils/api-utils.ts` consumes
  this envelope and surfaces toasts on `errors[]` / `warning*`.
- **No popups for non-destructive flows.** Inline UI, dedicated pages, and
  toasts by default; popups only for destructive confirmations.
- **Centralized brand tokens** in `app/globals.css` — rebrand by editing the
  token values there; components never hardcode colors.
- **Form logic in hooks.** Each form module ships `hooks/use-<module>-form.ts`
  owning useForm + onSubmit; components stay presentational.

See the root `CLAUDE.md` for the full rule set.

## Layout

```
app/
├── (auth)/            login / forgot / reset / change-password / verify-email
├── actions/           server actions (auth cookies)
├── context/           AuthProvider + VersionProvider + UnsavedChangesProvider
├── dashboard/         shell + (modules)/users + (modules)/settings
├── download/[...key]/ authenticated streaming download proxy
├── globals.css        Tailwind v4 + brand tokens (single source of truth)
└── layout.tsx         root layout, fonts, providers
components/
├── ui/                shadcn primitives
├── custom/            Form* field kit, CustomCard, FormFooter, tooltips, badges
├── table/             DataTable system (toolbar, filters, pagination, hooks)
└── data-grid/         editable spreadsheet-style grid
hooks/                 useUnsavedChangesWarning, useDataGrid, …
lib/
├── routes.ts          route constants (single source of truth)
├── enum.ts            numeric enums mirrored from the backend
├── permissions.ts     frontend permission matrix
├── nav-config.ts      role-aware sidebar config
├── app-settings.ts    the ONLY place NEXT_PUBLIC_* env vars are read
├── utils.ts           cn()
└── utils/             api-utils (envelope call()), format, zod-schema, …
```

## Getting started

```bash
cp .env.example .env   # fill NEXT_PUBLIC_SERVER_URL etc.
npm ci
npm run dev            # http://localhost:3000
```

## Scripts

- `npm run dev` — dev server (Turbopack)
- `npm run build` — production build
- `npm run lint` — eslint
- `npm test` — vitest (logic tests in `lib/**` + `app/**`)

Before committing: `npm run lint && npm run build && npm test` — CI does not
run these for you.

## Auth

The browser never talks to Cognito. Sign-in, renewal and sign-out all go
through this server, which calls the API and relays its session cookie onto
this origin (`lib/auth/session-bridge.ts`) — so there is no pool id, client id
or Cognito SDK on the client at all.

Two HttpOnly cookies: `token` (the Cognito access token, sent as Bearer) and
`sessionId` (an opaque handle the API trades for a new access token). The
proxy renews only when the access token has actually expired, and
`lib/auth/refresh-coordinator.ts` keeps concurrent RSCs to a single renewal.

A Cognito access token carries no identity claims, so `lib/auth/session-user.ts`
reads the signed-in user from the API rather than decoding the token.
