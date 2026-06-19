# EmailIntel — Deployment Guide

This bundle contains a fully wired-up version of your app:

- `index.html` — the frontend (fixed several bugs, see "What I fixed" below)
- `supabase/schema.sql` — database tables
- `supabase/functions/create-payment-intent/` — Stripe charge creation
- `supabase/functions/create-paypal-order/` — PayPal order creation
- `supabase/functions/capture-paypal-order/` — PayPal payment capture
- `supabase/functions/send-report/` — **real OSINT scan + email delivery**

---

## 1. Database setup

In your Supabase project → SQL Editor, run `supabase/schema.sql`. This creates the `leads` and `payments` tables referenced by the frontend.

## 2. Install the Supabase CLI (if you haven't)

```bash
npm install -g supabase
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

## 3. Set secrets (server-side only — never put these in index.html)

```bash
supabase secrets set STRIPE_SECRET_KEY=sk_test_...      # from Stripe Dashboard
supabase secrets set PAYPAL_CLIENT_ID=...               # from developer.paypal.com
supabase secrets set PAYPAL_SECRET=...
supabase secrets set PAYPAL_BASE_URL=https://api-m.sandbox.paypal.com   # switch to https://api-m.paypal.com when live
supabase secrets set RESEND_API_KEY=re_...               # from resend.com
supabase secrets set RESEND_FROM=reports@yourdomain.com  # must be a verified sending domain in Resend
```

## 4. Deploy the functions

```bash
supabase functions deploy create-payment-intent
supabase functions deploy create-paypal-order
supabase functions deploy capture-paypal-order
supabase functions deploy send-report
```

## 5. Verify your sending domain in Resend

Go to resend.com → Domains → add and verify your domain (DNS records). Until this is done, emails will fail to deliver to most inboxes.

## 6. Update the frontend CONFIG (already done, but double-check)

In `index.html`, the `CONFIG` block near the top holds:
- `GOOGLE_CLIENT_ID` — public, safe to expose
- `STRIPE_PUB_KEY` — public, safe to expose (this is the *publishable* key, not the secret key)
- `PAYPAL_CLIENT_ID` — public, safe to expose
- `SUPABASE_URL` / `SUPABASE_ANON_KEY` — public, safe to expose (protected by Row Level Security)

These are all meant to be public. The dangerous keys (`STRIPE_SECRET_KEY`, `PAYPAL_SECRET`, `RESEND_API_KEY`) live only in Supabase secrets, never in the HTML.

## 7. Host it

Any static host works: Netlify, Vercel, Cloudflare Pages, GitHub Pages, or Supabase Storage. Just upload `index.html`.

---

## What I fixed

1. **OSINT was a no-op** — `send-report` now actually queries GitHub's API, Gravatar, and checks ~18 platforms (Twitter/X, Reddit, Instagram, TikTok, LinkedIn, npm, PyPI, etc.) for the existence of a matching username, then builds a real HTML report and sends it via Resend.
2. **Stripe demo-mode detection bug** — the original check (`startsWith('pk_live_YOUR')`) never matched a real-looking test key, so it silently entered a broken state. Now checks for the literal placeholder string `YOUR` and falls back to demo mode cleanly, with a try/catch around Stripe init.
3. **Google Sign-In had no fallback** — One Tap is frequently blocked by browsers (Safari, incognito, ad-blockers). Added an OAuth popup fallback using `google.accounts.oauth2`.
4. **Duplicate event listeners** — `initStripe()` runs every time a user signs in; it was calling `addEventListener` on the pay button each time, stacking duplicate handlers and double-charging cards on repeat sessions. Switched to `.onclick =`.
5. **Duplicate PayPal SDK injection** — same root cause; added a `_paypalRendered` guard so the SDK script and buttons only render once per page load.
6. **Payment provider mislabeled** — `onPaymentSuccess` always logged `provider: 'stripe'` to the database regardless of actual payment method. Now correctly detects `demo` / `stripe` / `paypal` from the order ID shape.
7. **Stale inline setup comment** — the old HTML comment described Edge Functions that didn't match what was actually needed (it was missing PayPal entirely, and the `send-report` example was just a TODO stub). Replaced with accurate instructions pointing to this README.

## Security note

Your uploaded file had what look like **live-shaped API keys already hardcoded** (Google Client ID, Stripe publishable key, PayPal client ID, Supabase anon key). Publishable/anon keys are *designed* to be public, so that part is fine — but double check:
- The Stripe key is a `pk_test_...` (test mode) — good for now, swap to `pk_live_...` only when ready to take real money.
- Your Supabase RLS policies (in `schema.sql`) only allow inserts, not reads — confirm this is applied before going live, or anyone with your anon key could read all your leads/payments data.
