# PW-K8 CRM — Complete Technical Audit

**Audit date:** 2026-11-13  
**Audited baseline:** `main`  
**Working branch:** `audit/complete-crm-review`  
**Scope:** Static review of the repository files and branch history available through GitHub. No live production login, database, Firestore rules, Supabase policies, or deployment runtime was accessed.

## Executive summary

The repository is a single-page React application embedded in `index.html`. The current `main` branch is not safe to treat as a production CRM authentication boundary: its login form does not authenticate a user. It sets a local React state flag. The data layer is Firebase/Firestore with anonymous Firebase authentication when runtime-injected configuration exists, and sample in-memory data when it does not.

A separate branch, `feature/supabase-email-otp-login`, contains Supabase email OTP sign-in. It does **not** migrate the CRM data layer: Firebase imports, anonymous Firebase authentication, Firestore reads/writes, and the local mock-data fallback remain. Do not merge or deploy that branch as a complete authentication-and-data security fix without resolving the identity/data authorization boundary.

No changes have been made to `main` or the existing OTP feature branch.

## Repository inventory

- `index.html` — application source (React 18 UMD, Babel Standalone, Tailwind CDN, Firebase modular SDK).
- `README.md` — project overview and deployment notes.
- Default branch: `main`.
- A feature branch exists: `feature/supabase-email-otp-login`.
- One open GitHub item tracks Supabase email OTP sign-in.

## Findings

### Critical — Main branch login is not authentication

In `main/index.html`, the login form accepts an email address but does not validate credentials, request an OTP, or establish a server-backed session. Its submit handler calls `setIsLoggedIn(true)`. Anyone who can open the public site can enter the dashboard.

**Required fix:** use a real authentication provider, restore and monitor its session, enforce authorization on the data service, and derive the signed-in identity from the verified session. Hiding the UI is not authorization.

### Critical — Data access is not tied to the Supabase OTP identity

The OTP feature branch authenticates through Supabase, but the data layer still signs into Firebase anonymously and reads/writes Firestore collections. A Supabase user session does not automatically authorize or scope those Firebase operations.

**Required fix:** choose one authoritative identity and data authorization design. Preferred path, if the intended backend is Supabase: migrate CRM tables and all CRUD/realtime operations to Supabase, enable Row Level Security, and define role-based policies. Do not simply add a client-side role check.

### High — Production can silently fall back to demo records

When Firebase runtime configuration is unavailable or initialization fails, the app seeds hard-coded sample SKUs and requests in React state. Mutations then exist only in memory and disappear after refresh. The interface can look populated even though it is not connected to persistent storage.

**Required fix:** distinguish demo mode from production. Production must show a blocking configuration/connection error and must never silently show mock records as live data.

### High — Firestore security cannot be verified from this repository

The source uses collections under `artifacts/{appId}/public/data/skus` and `requests`, and anonymous Firebase authentication. Firestore rules are not present in the repository. It is therefore not possible to verify whether anonymous users can read, create, update, or delete operational records.

**Required fix:** inspect the actual Firebase project rules and remove public/anonymous write access. If migrating to Supabase, enforce RLS on every table and test with both authorized and unauthorized accounts.

### High — No verified role or permission model

The UI presents a generic authorized state and exposes add, status-change, and delete actions to any user who passes the UI gate. The code does not establish Admin, Operations, ZM, or read-only permissions.

**Required fix:** define roles and allowed actions with the business owner; enforce the same permissions in database policies or trusted server-side functions.

### Medium — Dashboard chart is illustrative, not data-driven

The pipeline doughnut uses fixed percentage segments (40/30/20/10) rather than counts derived from request statuses. This can misrepresent actual operations.

**Required fix:** calculate each segment from actual status counts and handle empty data.

### Medium — Order IDs are generated with a random five-digit suffix

The request form uses `Math.random()` to generate `SP-ORD-xxxxx`. This does not guarantee uniqueness and can collide.

**Required fix:** generate IDs in the database using a sequence/identity or UUID-backed unique constraint, and display a human-readable order number.

### Medium — Destructive actions have no confirmation flow

SKU and request delete buttons call delete handlers immediately. There is no confirmation, soft-delete, or audit trail visible in the source.

**Required fix:** add a confirmation dialog, prefer archive/soft-delete for business records, and record actor/time/reason for changes.

### Medium — Status changes are unrestricted

The UI permits direct selection of any listed status. There is no transition validation, required-field check, or status-change history.

**Required fix:** define permitted transitions (including ZM approval/decline, operations processing, dispatch, delivery), validate them in a trusted layer, and retain status history.

### Medium — Single-file, CDN/Babel development-style delivery

React UMD, Babel Standalone, Tailwind CDN, and other dependencies load from third-party CDNs at runtime. There is no package manifest, lockfile, production build, dependency audit, or automated test suite in the repository root.

**Required fix:** move to a reproducible build with pinned dependencies and a lockfile; add linting, tests, and a deployment workflow. Consider a restrictive Content Security Policy after removing inline scripts/styles as needed.

### Documentation mismatch

The README advertises Google Sheets sync, Excel export, and courier tracking. The audited `main/index.html` does not contain corresponding Google Sheets, SheetJS, or courier API integrations. These should not be represented as working features until verified.

## Branch comparison

| Area | `main` | `feature/supabase-email-otp-login` |
|---|---|---|
| Login | UI-only local state | Supabase email OTP |
| Session restore | No | Yes |
| Logout | Local state only | Supabase sign-out |
| CRM data | Firebase/Firestore | Still Firebase/Firestore |
| Identity linked to data authorization | No | Not demonstrated |
| Offline/demo fallback | Yes | Yes |

The Supabase URL and publishable key in the feature branch are browser-side configuration values; a publishable key is intended for client use. Its safety depends on correctly configured database policies and must not be confused with a secret/service-role key. No secret key was intentionally copied into this audit.

## Required information before safe implementation

The repository alone does not expose the backend configuration needed to make a complete, safe data-layer migration. Before implementing the production data/security changes, confirm:

1. Whether PW-K8-CRM should use the Supabase project already referenced by the OTP feature branch, or a different dedicated project.
2. The intended user roles and permissions (for example Admin, Operations, ZM, and read-only).
3. The actual tables/columns and existing data that must be preserved, or a sanitized schema export.
4. Whether the Firebase project is still live and must be migrated, or is only legacy/demo infrastructure.

Never share service-role keys, Firebase private credentials, passwords, or OTPs. Public/publishable client keys are different from secret keys.

## Verification checklist for the implementation phase

- [ ] Main branch and current production deployment remain unchanged.
- [ ] OTP only works for pre-provisioned users; expired/invalid codes and rate limits are handled.
- [ ] Session persists after refresh and logout invalidates the session.
- [ ] Unauthorized users cannot read or mutate CRM records through direct API calls.
- [ ] Admin/Operations/ZM/read-only permissions are enforced by the backend.
- [ ] SKU create/update/archive and stock validation work against persistent storage.
- [ ] Order number uniqueness is guaranteed by the database.
- [ ] Request status transitions are validated and audited.
- [ ] Dashboard metrics match database counts.
- [ ] Production never falls back to mock data.
- [ ] Export, Google Sheets, and courier integrations are tested before being advertised.
- [ ] Responsive layout, empty/loading/error states, and accessibility are checked.
- [ ] Automated tests and a production build pass before a pull request is marked ready.

## Safety decision

This branch records the audit findings only. The critical authentication/data-boundary issues must not be "fixed" by blindly merging the OTP branch or by guessing database policies. The next implementation step is to confirm the backend and role model, then make the changes on this isolated branch and validate them before any merge.


## Additional connected Supabase project inspection

The Supabase URL embedded in the existing OTP feature branch resolves to the connected project named **rohitkumarprajapti1996@gmail.com's Project** (region `ap-southeast-1`). I inspected its public table metadata, migration history, edge-function inventory, and Supabase security/performance advisors using read-only operations. No database writes, migrations, function deployments, or policy changes were made.

The project currently contains CRM snapshot/access tables (including `crm_shared_snapshots`, `crm_printing_snapshots`, `crm_print_projection_workbooks`, and `crm_access_requests`) alongside several e-commerce/Shopify table families and payment-related functions. This confirms the project is not an empty, dedicated PW-K8-CRM database. It may be shared with other active workflows and must not be modified as part of this repository work without explicit confirmation.

### Live project advisor findings

- **RLS enabled but no policies:** Supabase reports three public tables with RLS enabled and no policies: `ecommerce_admins`, `shopify_admins`, and `shopify_lite_payment_webhook_events`. This is not proof of public exposure; with no policies, ordinary RLS-protected access is denied. Confirm intended server-side access before changing anything.
- **Callable SECURITY DEFINER functions:** advisors report public execution of `rls_auto_enable()` and `shopify_is_admin()` for anon, and authenticated execution of additional SECURITY DEFINER functions including `is_ecommerce_admin()` and `shopify_lite_create_order(...)`. These functions need source and grants reviewed. Do not revoke grants blindly because live commerce workflows may depend on them.
- **Auth hardening:** leaked-password protection is disabled according to the Auth advisor.
- **Performance:** advisors report 11 unindexed foreign keys, 12 RLS policies with per-row auth-function evaluation, and several overlapping permissive policies. These should be prioritized only after confirming table ownership and workload impact.

Relevant Supabase advisor references:
- [RLS enabled without policies](https://supabase.com/docs/guides/database/database-linter?lint=0008_rls_enabled_no_policy)
- [SECURITY DEFINER function executable by anon](https://supabase.com/docs/guides/database/database-linter?lint=0028_anon_security_definer_function_executable)
- [SECURITY DEFINER function executable by authenticated users](https://supabase.com/docs/guides/database/database-linter?lint=0029_authenticated_security_definer_function_executable)
- [Leaked password protection](https://supabase.com/docs/guides/auth/password-security#password-strength-and-leaked-password-protection)
- [Unindexed foreign keys](https://supabase.com/docs/guides/database/database-linter?lint=0001_unindexed_foreign_keys)
- [RLS init plan](https://supabase.com/docs/guides/database/database-linter?lint=0003_auth_rls_initplan)
- [Multiple permissive policies](https://supabase.com/docs/guides/database/database-linter?lint=0006_multiple_permissive_policies)

**Important:** the Supabase project inspection changes the implementation decision: do not apply schema changes or security-policy edits to this project yet. First confirm whether this is the intended backend for PW-K8-CRM and identify which existing tables/workflows must remain untouched.
