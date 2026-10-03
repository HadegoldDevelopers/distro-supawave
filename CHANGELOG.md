# Changelog

All notable changes to **Distro Supawave** are documented in this file.

## v1.1.0 — 2026-10-01

### Security & hardening
- Installer self-destruct (`/install/complete`) is **POST-only** and requires a
  completed installation + one-time session flag.
- Chunked audio uploads enforce a strict extension whitelist and verify the
  assembled file's MIME before storage.
- **Master audio files** are stored in **private storage** with unique UUID
  filenames and served only through authorized endpoints — never from the
  public web root.
- Release finalization verifies every `audio_path` and `cover_path` was
  uploaded by the current user in the current session.
- Ownership checks (IDOR) on releases, stats, and artists.
- Default Laravel middleware stack preserved; payment webhooks are CSRF-exempt.
- Installer requires PHP >= 8.2.

### Payments
- **All gateways create full transactions** (`original_amount`/`original_currency`)
  and every callback is **idempotent**: a single external payment can only
  activate a subscription once (atomic `pending → paid` transitions).
- **PayPal** — browser callback and webhook resolve and atomically claim the
  transaction (row-locked) before capture; an already-captured order is never
  captured again, and failed local activation is reconciled by checking the
  PayPal order status so retries converge.
- **Paystack** — pending transaction created at init; callback is idempotent
  with no duplicate paid records.
- **NOWPayments** — signature verified over recursively-sorted callback params;
  pending transaction created before redirect; unique order IDs.
- **CoinPayments** — HMAC + merchant ID verified; IPN secret configurable.
- **MoneyUnify** — atomic activation, null guards, idempotent.
- **Manual Payment** — transaction created once via POST with server-side
  amount/currency; admin approve/reject flow.
- Live conversion claim removed — rates are **admin-configurable**.

### Royalties & currency
- `massApprove()` is transaction-safe and idempotent.
- Seeded USD rate corrected to 1.00; zero rates guarded against division-by-zero.

### Documentation
- Accurate setup guides for NOWPayments IPN Secret and PayPal webhooks.
- No license/domain-binding claims anywhere.
- Version information made consistent across package, docs, and changelog (v1.1.0).

### PayPal failure recovery
- The external PayPal capture is performed **outside** any DB transaction or
  row lock (no lock is held during the HTTP call).
- A durable claim (`processing_at`) atomically prevents concurrent
  callbacks/webhooks from capturing twice; stale claims (> 5 min) are reclaimed.
- A successful capture is recorded durably (`captured_at`) so that even if the
  local activation step fails and rolls back, a later request (or PayPal webhook
  retry) completes activation **without re-capturing** the order.
- `captureOrder()` now sends a `PayPal-Request-Id` idempotency header and
  reconciles `ORDER_ALREADY_CAPTURED` responses.
- The PayPal access token is no longer written to the logs.
- Added migration `2026_10_02_000000` adding `processing_at`, `captured_at` and
  `attempts` columns to `transactions`.

## v1.0.0 — 2026-09-16

- Initial release: white-label music distribution platform for artists and labels.