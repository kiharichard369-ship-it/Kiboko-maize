# Kiboko Maize Ledger

A staff-only, mobile-first ledger for a maize flour posho mill. Maize deposits and flour pickups are recorded as append-only entries; customer balances are calculated from those entries, never edited as a stored number.

## What is included

- Staff email/password sign-in, owner/attendant authorization, and a clearly labeled first-run demo mode.
- Searchable customer tabs; deposits in kg or tins; sifted/unsifted flour pickups; live extraction, fee, and resulting-balance previews.
- In-kind and cash milling fees, cash/M-Pesa payment states, payment retries, and receipt visibility.
- Owner-only Today, Stock check, and Settings screens, including rate/tin settings and staff role management.
- Supabase migrations for profiles, customers, settings, an immutable ledger, balance view, M-Pesa attempts, RPCs, triggers, RLS, and Realtime; Supabase Edge Functions for Daraja STK push and callback.

The supplied starting values are placeholders and should be replaced with Kiboko's actual rates: unsifted extraction **0.95**, sifted extraction **0.75**, in-kind fee **10% of maize milled**, cash fee **KES 5 per flour kg**, and tin/gorogoro **2 kg**. Pickup rows snapshot the settings used so later changes do not rewrite history.

## Run the website locally or in Preview

Requirements: Node.js and pnpm (the project pins `pnpm@11.25.0`). From the project root:

```sh
pnpm install --frozen-lockfile
cp .env.example .env.local
pnpm dev
```

With the example's `VITE_DEMO_MODE=true`, the app opens with sample owner and attendant access, three synthetic customers, and sample ledger activity. Demo changes stay in that browser's local storage; they are not shared between devices and must not be used as business records. The interface displays a Demo mode indicator. When running the Vite development server without Supabase credentials, demo mode is also selected explicitly as a development fallback.

For production mode, supply the browser-safe Supabase URL and anon/publishable key and set `VITE_DEMO_MODE=false`. A production build without configured Supabase credentials shows setup/sign-in guidance instead of accepting demo writes. Never place a service-role key, Daraja consumer secret, or passkey in a `VITE_` variable.

```sh
pnpm typecheck
pnpm build
```

## Configure Supabase

1. Create a Supabase project and install the Supabase CLI. In `Authentication → Providers`, enable Email/password and turn off public sign-ups. Staff accounts are provisioned by an administrator in Supabase Auth; there is no customer login or public staff-registration screen.
2. Link the CLI to the project and apply the migration:

   ```sh
   supabase login
   supabase link --project-ref <project-ref>
   supabase db push
   ```

   The migration creates the default mill settings and an Auth-user trigger that creates each new staff profile as an `attendant`. `supabase/seed.sql` is an idempotent default-settings seed for manual/local use; do not seed invented customer transactions into a real mill database.
3. Create the first owner account through Supabase Auth after the migration is applied. The Auth trigger creates its attendant profile. In the Supabase SQL Editor, promote that account by its email using the trusted admin connection:

   ```sql
   update public.profiles
   set role = 'owner'
   where id = (
     select id from auth.users where lower(email) = lower('owner@example.com')
   );
   ```

   Replace the example email with the actual owner account. Keep at least one owner; the role-management RPC prevents demoting the last owner. Create other staff accounts in Supabase Auth; they appear here as attendants, and an owner can change roles in Settings.
4. Copy `.env.example` to `.env.local` and set the project's browser-safe values from Supabase **Project Settings → API**:

   ```dotenv
   VITE_SUPABASE_URL=https://<project-ref>.supabase.co
   VITE_SUPABASE_ANON_KEY=<publishable-or-anon-key>
   VITE_DEMO_MODE=false
   ```

   Never use the service-role/secret key in this file. Set the allowed Auth redirect URLs in Supabase if the project or deployment domain changes.
5. Confirm the Realtime publication contains `public.customers`, `public.ledger_entries`, and `public.mpesa_payment_attempts`. The migration adds these to Supabase's standard `supabase_realtime` publication if it exists. TanStack Query invalidates the relevant tabs, totals, and owner reconciliation queue when these tables change.

### Ledger and access controls

`public.customer_balances` is a security-invoker view that sums `ledger_entries.maize_kg_delta`. Deposit and pickup rows are created by restricted RPCs, which record the authenticated staff profile and timestamp. A transaction-level per-customer lock protects the insufficient-balance check against simultaneous pickups and blocks a deposit reversal if it would drive the tab below zero. Triggers reject staff-facing edits and all deletes to ledger entries, invalid conversion/fee arithmetic, and duplicate reversals. The only ledger `UPDATE` exceptions are narrow M-Pesa payment-metadata changes by service-role functions, and an audited owner-only reconciliation RPC after the provider result is verified. These paths cannot change quantities, fees, customer, actor, or timestamp. Owners correct a mistake by adding one reason-required inverse row; the original remains visible. Reversing a paid cash-fee pickup asks whether the fee was retained or partially/full refunded by cash or M-Pesa; M-Pesa refunds require a recorded provider reference. Today shows gross collected fees and recorded refunds separately, even after a tab reversal: cash collection is recorded with the pickup, while M-Pesa collection is reported on the mill day when Daraja confirms or the owner reconciles it. RLS protects every application table; attendants cannot edit settings, manage staff, or create reversals. No application path hard-deletes customer, staff, settings, ledger, or payment-attempt records.

## Configure M-Pesa Daraja

M-Pesa is optional until Kiboko has its Daraja shortcode/till, passkey, and consumer credentials. The browser only calls the authenticated `mpesa-stk-push` function; Daraja credentials and the callback token are Edge Function secrets.

1. Set the function secrets in the linked project. Supabase provides `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` to Edge Functions at runtime; verify those standard runtime values are available. Set the Daraja values (sandbox/live base URL must match the credentials):

   ```sh
   supabase secrets set \
     DARAJA_BASE_URL=<Daraja-sandbox-or-live-base-url> \
     DARAJA_CONSUMER_KEY=<consumer-key> \
     DARAJA_CONSUMER_SECRET=<consumer-secret> \
     DARAJA_SHORTCODE=<business-shortcode> \
     DARAJA_PASSKEY=<STK-passkey> \
     DARAJA_CALLBACK_URL=https://<project-ref>.supabase.co/functions/v1/mpesa-callback \
     DARAJA_CALLBACK_TOKEN=<random-secret-token>
   ```

   Generate a high-entropy callback token (for example, 32 random bytes encoded as hex) and keep it only in the function secret store. The STK function adds it as a query token to the callback URL; the callback validates it before processing a callback. Do not paste real credentials into Git, browser variables, screenshots, or chat.
2. Deploy both functions:

   ```sh
   supabase functions deploy mpesa-stk-push
   supabase functions deploy mpesa-callback
   ```

   `supabase/config.toml` requires a staff JWT for STK push and disables Supabase JWT verification only for the provider callback; the callback's private high-entropy token is checked in the function. Point Daraja at the exact HTTPS callback URL above. Rotate the callback token if it is exposed.
3. In Settings, replace the placeholder rates and tin size with Kiboko's values. The fee calculation is kept at the exact decimal value in the ledger. Daraja STK requests accept whole KES, so the function rounds the provider amount to the nearest shilling and returns both the exact ledger due and provider amount for reconciliation. Configure rates with this in mind.
4. M-Pesa pickups stay visibly `PENDING` until a provider result is confirmed and, on success, the receipt is recorded. A staff member can use **Retry M-Pesa** from the tab after a confirmed failure. The server blocks a duplicate prompt while an STK request is active. If no callback arrives within two minutes—or the request outcome is ambiguous or timed out—the request appears in owner-only **M-Pesa reconciliation** in Settings; the app never launches a second prompt automatically. Every unresolved state, including `TIMED_OUT`, blocks both retries and reversals until the owner checks Daraja. Check transaction history, choose the verified result, enter a provider receipt when paid, and record the verification reason. Only then is the ledger payment state resolved and a confirmed failure can be retried. Late callbacks are matched to their attempt and processed idempotently. If multiple attempts report paid, all receipts remain visible in the customer ledger and the owner is warned to follow up on the duplicate charge.

## Production build and deployment notes

The frontend is a static Vite build backed by Supabase; it does not need a Manus-managed database or application server. `pnpm build` writes `dist/`. For a static host, serve `index.html` as the SPA fallback for `/login`, `/customers/:customerId`, `/today`, `/stock`, and `/settings`. Keep the `VITE_` values to the Supabase URL and anon/publishable key only. Apply the migration and deploy Edge Functions separately in the Supabase project.

## Product boundaries

This v1 is for staff counter operations; it has no public marketing pages, no customer login, no walk-in flour sale that bypasses a customer tab, and no Africa's Talking SMS receipts. Offline state is announced clearly and all business writes are blocked while offline; the app never silently queues an entry.
# Kiboko-maize
