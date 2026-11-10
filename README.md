# LINK-SMART

Multi-business management platform built with **React + TypeScript + Vite + Tailwind CSS + Firebase**.

One deployment serves four business verticals with strict data isolation:

| Business type | What it manages |
| --- | --- |
| **Shop** | Products, categories, stock, POS sales, purchases, suppliers, customers, debts |
| **Hotel** | Rooms, room categories, reservations, guests, check-in/out, hotel services, hotel restaurant |
| **Restaurant / Cafe** | Menu, food categories, tables, orders, kitchen board, POS |
| **Fuel station** | Fuel types, prices, tanks, pumps, meter readings, fuel purchases and sales |

A separate **Super Admin console** manages businesses, users, custom roles, platform reports, audit logs and global settings.

---

## 1. Architecture

- **Frontend**: single-page React app (Vite), lazy-loaded routes, Tailwind design system in `src/components/ui`.
- **Auth**: Firebase Authentication (email/password). Super Admin is identified **only** by the custom claim `superAdmin === true` — never by email at runtime.
- **Database**: Cloud Firestore. Every business document carries `businessId`; Security Rules enforce tenant isolation and per-module permissions.
- **Storage**: Firebase Storage for business logos and user avatars (images < 2 MB).
- **Backend**: Cloud Functions (callable) for privileged user management and Super Admin bootstrap.
- **Hosting**: Netlify (SPA redirects already configured).

```
src/
  components/      UI library, layout shell, generic ResourcePage + DataTable
  config/          CRUD resource definitions (fields, columns, side effects)
  context/         Auth, Theme, Language, Toast providers
  firebase/        config, audit helper, admin user helpers
  hooks/           tenant-scoped and admin Firestore subscriptions
  lib/             permissions, constants, formatting, errors, i18n
  pages/           auth, business modules, super admin console
functions/         Cloud Functions (node 20)
firestore.rules    Tenant isolation + permission enforcement
storage.rules      Logo/avatar upload rules
firestore.indexes.json
```

---

## 2. Prerequisites

- Node.js 20+ (Node 24 works)
- npm 10+
- Firebase CLI: `npm i -g firebase-tools` (or `npx firebase-tools`)
- A Firebase project with **Authentication (Email/Password)**, **Firestore**, **Storage** and **Functions** enabled

---

## 3. Environment variables

Copy the example file and edit it:

```powershell
Copy-Item .env.example .env
```

```env
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_DATABASE_URL=https://your-project-default-rtdb.firebaseio.com
VITE_FIREBASE_PROJECT_ID=your-project
VITE_FIREBASE_STORAGE_BUCKET=your-project.firebasestorage.app
VITE_FIREBASE_MESSAGING_SENDER_ID=...
VITE_FIREBASE_APP_ID=...
VITE_FIREBASE_MEASUREMENT_ID=...

VITE_SUPER_ADMIN_EMAIL=ishanews7@gmail.com
```

- These `VITE_*` values are **public client configuration** and safe to ship in the browser. Blank values fall back to the built-in LINK-SMART project config in `src/firebase/config.ts`.
- `.env.local` is optional and gitignored; Vite loads it after `.env` (useful for machine-specific overrides).
- `VITE_SUPER_ADMIN_EMAIL` is enforced in the client and in `firestore.rules` (email fallback). It must match the Cloud Function env var `SUPER_ADMIN_EMAIL` (`functions/.env`) when the claim bootstrap is used.
- Never put the Firebase **service account key** or Admin SDK private key in the frontend or in `VITE_*` variables.

---

## 4. Install

```powershell
npm install --legacy-peer-deps
```

If the registry rejects `@babel/parser` during a full install, install the runtime packages explicitly:

```powershell
npm install firebase@10.14.1 recharts@2.12.7 lucide-react@0.451.0 --save-exact --legacy-peer-deps
npm install -D typescript@5.6.3 tailwindcss@3.4.17 postcss@8.4.49 autoprefixer@10.4.20 @types/react@18.3.12 @types/react-dom@18.3.1 --save-exact --legacy-peer-deps
```

If TypeScript reports that Firebase modules "have no exported members", the installed `@firebase/*` packages are incomplete. Reinstall the affected packages (they ship their `.d.ts` files):

```powershell
npm install firebase@10.14.1 @firebase/auth@1.7.9 --save-exact --legacy-peer-deps --prefer-offline
```

---

## 5. Firebase project setup

### 5.1 Enable services

In the Firebase Console:

1. **Authentication → Sign-in method → Email/Password → Enable**
2. Create **Firestore Database** (production mode)
3. Create **Storage** bucket
4. Upgrade to the Blaze plan to deploy **Cloud Functions**

### 5.2 Deploy rules, indexes and functions

```powershell
firebase login
firebase use your-project-id

# Rules + composite indexes (indexes may take a few minutes to build)
firebase deploy --only firestore:rules,firestore:indexes,storage

# Cloud Functions
cd functions
npm install
cd ..
firebase deploy --only functions
```

> The app works before indexes finish building: `useCollection`/`useAdminCollection` automatically fall back to an unordered query when Firestore returns `failed-precondition`.

### 5.3 Configure the Super Admin email on the server

The Cloud Function only grants the `superAdmin` claim to the configured address.

**Option A — functions `.env` (recommended for Firebase Functions v2+ / local emulator):**

```powershell
Copy-Item functions\.env.example functions\.env
# then edit:
# SUPER_ADMIN_EMAIL=ishanews7@gmail.com
```

**Option B — legacy runtime config:**

```powershell
firebase functions:config:set app.super_admin_email="ishanews7@gmail.com"
```

Redeploy functions after changing the value:

```powershell
firebase deploy --only functions
```

### 5.4 Create the Super Admin account

Two supported methods:

**A. In-app bootstrap (recommended)**

1. Open `/setup`
2. Enter the exact `VITE_SUPER_ADMIN_EMAIL` address and a password (min 6 chars)
3. The page creates/signs in the account on a throw-away secondary Firebase app, calls `bootstrapSuperAdmin`, refreshes the token, then signs you in on the main app and redirects to `/app/admin`

**B. Offline script**

```powershell
# functions/set-super-admin.js — requires functions/serviceAccountKey.json
node functions/set-super-admin.js ishanews7@gmail.com
```

If the Auth account does not exist yet, the script can create it. The password is read from an environment variable and is never stored in the repo:

```powershell
$env:SUPER_ADMIN_PASSWORD="your-password"; node functions/set-super-admin.js ishanews7@gmail.com
```

The account must sign in again (or refresh its token) after the claim is set. The in-app bootstrap handles this automatically.

### 5.5 First-run data flow

1. Sign in as Super Admin → `/app/admin`
2. **Businesses → New business** — create the tenant and choose its type
3. **Users → Create user** — assign the user to that business, pick a role, adjust permissions
4. The business user signs in and lands on their type-specific dashboard

---

## 6. Run locally

```powershell
npm run dev        # Vite dev server
npm run typecheck  # TypeScript, no emit
npm run build      # production build to dist/
npm run preview    # preview the production build
```

Optional: emulate Firebase locally.

```powershell
firebase emulators:start --only auth,firestore,functions,storage
```

---

## 7. Deploy to Netlify

Netlify hosts **only the Vite frontend**. The backend is Firebase (Auth, Firestore, Storage, Cloud Functions), called directly from the browser with the Firebase SDK. **No Netlify Functions are required or used.** `netlify.toml` is a single valid TOML file with `NODE_VERSION = "20"` defined once, `npm run build` / `dist`, and an SPA redirect (`/* → /index.html 200`).

1. Push the repository to GitHub/GitLab (do **not** commit `.env` or `functions/.env`)
2. Netlify → **Add new site → Import from Git**
3. Netlify reads `netlify.toml` automatically:
   - Build command: `npm run build`
   - Publish directory: `dist`
   - Node: `20`
4. Optional but recommended — add these in **Site configuration → Environment variables**:
   `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_DATABASE_URL`,
   `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`,
   `VITE_FIREBASE_MESSAGING_SENDER_ID`, `VITE_FIREBASE_APP_ID`,
   `VITE_FIREBASE_MEASUREMENT_ID`, `VITE_SUPER_ADMIN_EMAIL`.
   If omitted, the app falls back to the built-in LINK-SMART web config in `src/firebase/config.ts` (public values only).
5. Deploy

Firebase backend must be configured once (it is **not** deployed by Netlify):

```powershell
firebase deploy --only firestore:rules,firestore:indexes,storage,functions
```

Post-deploy checklist:

- `/login`, `/setup`, `/app/*` load directly on refresh (SPA redirects)
- Super Admin can sign in and load `/app/admin`
- A business user sees only their own business data
- Logo/avatar uploads succeed (Storage rules)
- Cloud Functions show as deployed in the Firebase Console (optional with the email fallback)
- No `[build.environment]` duplication error in the Netlify deploy log

---

## 8. Roles and permissions

Permissions are defined per business type in `src/lib/permissions.ts`. A user document stores:

```json
{
  "fullName": "Amina Yusuf",
  "email": "amina@example.com",
  "businessId": "businessDocumentId",
  "businessType": "restaurant",
  "role": "Cashier",
  "permissions": ["dashboard", "pos", "orders", "customers", "payments"],
  "status": "active"
}
```

- The sidebar shows only modules the user has permission for.
- `PermissionRoute`/`Guard` blocks direct URL access on the client.
- **Firestore Rules enforce the same permissions on reads and writes** — client-side hiding is not the security boundary.

Roles include Business Owner, General Manager, Administrator, Accountant, Cashier, Sales Manager, Inventory Manager, Employee, plus reusable **Custom Roles** stored in the `roles` collection.

---

## 9. User administration

The Super Admin can:

- **Create users** (secondary Firebase app — your session is never replaced)
- **Edit** profile, business, role and individual permission checkboxes
- **Enable/disable** accounts (Firestore status + Firebase Auth record)
- **Reset passwords** (set a new password, or send a reset email)
- **Delete users** (Auth account + Firestore profile)

Privileged operations call the Cloud Functions `adminSetDisabled`, `adminResetPassword` and `adminDeleteUser`. If the functions are not deployed, the app falls back to Firestore-only changes and shows a warning.

All privileged actions are written to the `auditLogs` collection and visible under **Audit Logs**.

---

## 10. Data model (main collections)

`businesses`, `users`, `roles`, `businessSettings`, `settings`, `notifications`, `auditLogs`,
`products`, `categories`, `inventory`, `sales`, `orders`, `purchases`, `suppliers`, `customers`,
`employees`, `expenses`, `debts`, `payments`, `rooms`, `roomCategories`, `reservations`, `guests`,
`hotelServices`, `tables`, `menuItems`, `fuelTypes`, `fuelTanks`, `fuelPumps`, `meterReadings`,
`fuelSales`.

Business records store numeric timestamps (`createdAt`, `updatedAt`) and `businessId` for tenant scoping. POS sales also create a `payments` record so revenue reporting stays consistent.

---

## 11. Security notes

- The Admin SDK **must only** run inside Cloud Functions; it is never bundled with the frontend.
- Super Admin access requires the `superAdmin` custom claim, **or** the single owner address `ishanews7@gmail.com` (email fallback configured in `AuthContext` and `firestore.rules`). Editing a `users` document cannot grant either. For multi-admin or hardened setups, remove the email fallback and rely on the claim only.
- Rules prevent changing `businessId` (tenant hopping) and prevent self-editing of role, permissions, status, business type or business ID.
- Storage writes are limited to images under 2 MB; user avatars can only be written by the matching authenticated user.
- Privileged audit events are created with `actorUid` equal to the caller and are readable only by the Super Admin.

---

## 12. Troubleshooting

| Symptom | Fix |
| --- | --- |
| Blank page / "Firebase is not configured" | Check `.env` values and restart `npm run dev` |
| `The backend function is not deployed yet` | Run `firebase deploy --only functions` |
| Lists empty with `failed-precondition` | Deploy indexes: `firebase deploy --only firestore:indexes` |
| Setup says `SUPER_ADMIN_EMAIL is not configured` | Set the functions env var and redeploy functions |
| Setup says `permission-denied` | The entered email must exactly match the server `SUPER_ADMIN_EMAIL` |
| Uploads fail | Verify `storage.rules` is deployed and the file is an image under 2 MB |
| TypeScript cannot find Firebase exports | Reinstall Firebase packages (see §4) |

---

## 13. License

Private project. All rights reserved.
