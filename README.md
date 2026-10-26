# Runbhoomi Ticket Booking System

A modular Django ticket-booking application with event discovery, customer accounts, inventory holds, sandbox or Stripe checkout, e-tickets with QR codes, and a staff dashboard.

## Requirements

Python 3.10+ (developed with Python 3.13), pip, and optionally PostgreSQL. SQLite is used when `POSTGRES_DB` is unset.

## Local setup

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
python manage.py migrate
python manage.py seed_demo
python manage.py createsuperuser
python manage.py runserver
```

Visit `http://127.0.0.1:8000/`. The sandbox payment is clearly labelled and never takes a real payment. Create a staff user with `createsuperuser` to access `/admin-dashboard/` and `/admin/`.

## Booking behavior

Prices and tax are calculated server-side. Event and ticket rows are locked with `select_for_update` during reservation; active holds and confirmed sales both count against inventory. A hold expires after `BOOKING_HOLD_MINUTES`; run `python manage.py expire_bookings` periodically to mark expired holds and fail their pending payments. Sandbox confirmations are transactional and idempotent. Optional Stripe mode creates server-side PaymentIntents and only confirms tickets after a verified `payment_intent.succeeded` webhook. Customer views scope bookings and tickets to the signed-in owner.

SQLite does not provide PostgreSQL's row-level locking guarantees. Use PostgreSQL in production by setting `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_HOST`, and `POSTGRES_PORT`.

## Configuration and deployment

Copy `.env.example` to `.env` and set a long random `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=0`, and the production `DJANGO_ALLOWED_HOSTS`. Set `DJANGO_SECURE_SSL_REDIRECT=1` behind correctly configured HTTPS. PostgreSQL credentials and payment secrets belong only in environment variables or a deployment secret manager. Set `PAYMENT_MODE=mock` only for local development. The checkout provider boundary is `bookings.services.confirm_mock_payment`; configure a provider adapter and signature-verified webhook before enabling live charges. Card details are never collected or stored by Runbhoomi.

For production, set `DJANGO_DEBUG=0`, a long random secret, `DJANGO_SECURE_SSL_REDIRECT=1`, and enable HSTS with `DJANGO_SECURE_HSTS_SECONDS` only after HTTPS is confirmed. Set `PAYMENT_MODE=stripe`, `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, and `STRIPE_WEBHOOK_SECRET`; configure Stripe to post `payment_intent.succeeded` and `payment_intent.payment_failed` to `/payments/stripe/webhook/`. Run `python manage.py check --deploy`, collect static files with `python manage.py collectstatic`, serve media from protected object storage, configure trusted proxy headers for your hosting platform, and schedule `expire_bookings` at least once per minute. Configure an SMTP email backend before enabling password-reset emails. The app uses Django's built-in admin plus a separate staff-only dashboard. `is_staff` is required on every dashboard endpoint.

## Tests and sample events

```powershell
python manage.py test
python manage.py seed_demo
```

The booking tests cover server-side pricing, inventory holds, expiry release, idempotent payment confirmation, QR ticket generation, and customer object permissions. Demo data uses generic generated event descriptions and CSS art; upload event imagery through Django admin for production presentation.

## Project layout

`accounts` manages email-based accounts, `events` manages the event catalogue, `bookings` owns reservation and ticket services, `payments` stores payment and webhook idempotency records, `core` provides contact/audit models and public pages, and `dashboard` contains staff-only operations and CSV reporting.
