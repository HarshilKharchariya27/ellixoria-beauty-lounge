# Ellixoria Beauty Lounge

Luxury salon booking and product shopping for Ellixoria Beauty Lounge in
Nadiad, Gujarat. The app uses Next.js App Router, React, strict TypeScript,
Tailwind CSS v4, Firebase Authentication, Cloud Firestore, and Cloud Storage.

## Requirements

- Node.js 24 LTS and npm.
- A Firebase project for live authentication, bookings, orders, and admin tools.
- Firebase CLI for local emulators and deploying rules/indexes.
- Chromium for Playwright browser tests.

## Local development

```powershell
Copy-Item .env.example .env.local
npm install
npm run dev
```

Open `http://localhost:3000`. Public pages render setup states without Firebase
credentials; authentication, bookings, shopping, and admin operations require
Firebase configuration.

### Firebase web and server configuration

In Firebase Console:

1. Create a project and register a web app. Copy its web configuration to the
   `NEXT_PUBLIC_FIREBASE_*` variables in `.env.local`.
2. Enable Email/Password, Google, and Phone providers under Authentication.
   Configure the Google OAuth consent screen and authorized domains. Phone
   sign-in requires a real authorized domain, reCAPTCHA, quota, and may incur
   SMS charges; use Firebase test phone numbers during development.
3. Create the Firestore database and a Cloud Storage bucket in the same project.
4. Use a server-only service account or Application Default Credentials for
   the Admin SDK. For a local service account, set
   `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, and
   `FIREBASE_ADMIN_PRIVATE_KEY`. Store the private key securely; `.env.local`
   is ignored by Git and must never be deployed as a file or committed.
5. Set `NEXT_PUBLIC_SITE_URL` to the absolute public origin (for example,
   `https://your-domain.example`) before production deployment. Local
   development uses `http://localhost:3000` when this is unset. Set
   `NEXT_PUBLIC_INSTAGRAM_URL` only if the lounge has a confirmed profile URL.

The public web configuration and Admin SDK credentials serve different
purposes. Never expose Admin credentials through a `NEXT_PUBLIC_` variable.

### Firebase SQL Connect schema

An initial Firebase SQL Connect GraphQL schema for the salon's users, catalog,
staff, bookings, orders, wishlists, reviews, and site settings is in
`dataconnect/schema/schema.gql`. It models a new PostgreSQL database; it does
not connect the existing application to SQL Connect, copy Firestore records,
or replace Firestore. The running app continues to use Firestore.

Before deploying SQL Connect, install/update Firebase CLI, run
`firebase init dataconnect`, select the correct project, region, and Cloud SQL
instance, and review the generated `dataconnect/dataconnect.yaml` and
`firebase.json` configuration. Then validate the schema with the SQL Connect
emulator and CLI. SQL Connect requires a Cloud SQL for PostgreSQL instance and
may incur charges. A deliberate data migration and application integration
would be separate work; do not deploy a schema migration until its generated
database changes have been reviewed.

### Firebase emulators

Install the Firebase CLI if it is not available:

```powershell
npm install --global firebase-tools
firebase login
```

Use `demo-ellixoria` as the emulator project ID or replace it with your Firebase
project ID. For a demo project, configure the matching ID in `.env.local` and
start the emulators:

```powershell
$env:NEXT_PUBLIC_USE_EMULATORS = "true"
$env:NEXT_PUBLIC_FIREBASE_PROJECT_ID = "demo-ellixoria"
$env:FIREBASE_ADMIN_PROJECT_ID = "demo-ellixoria"
$env:FIRESTORE_EMULATOR_HOST = "127.0.0.1:8080"
$env:FIREBASE_AUTH_EMULATOR_HOST = "127.0.0.1:9099"
$env:FIREBASE_STORAGE_EMULATOR_HOST = "127.0.0.1:9199"
npm run emulators -- --project demo-ellixoria --only auth,firestore,storage
```

Leave that terminal running. In a second terminal, use the same environment
variables and run `npm run dev`. The Firebase web settings need all six
`NEXT_PUBLIC_FIREBASE_*` values; emulator credentials can be placeholder
values, but the project ID must match. Do not use production credentials when
testing against emulators.

The Firestore rules tests use `FIRESTORE_EMULATOR_HOST`. They skip when the
Firestore emulator is not running. The app itself does not seed demo products,
services, staff, or reviews.

### Seed the salon and create an admin

After setting server credentials and choosing the intended Firebase project:

```powershell
npm run seed
npm run set-admin -- harshilkharchariya27@gmail.com grant
npm run set-admin -- beautyloungeellixoria@gmail.com grant
```

The seed script creates default site settings and service categories only; it
does not replace existing settings or invent catalog content. Create the user
through Firebase Authentication first, then grant the admin claim. Only
`harshilkharchariya27@gmail.com` and `beautyloungeellixoria@gmail.com` are
allowlisted for admin access. Sign out and sign in again to refresh the user's
token. To revoke access:

```powershell
npm run set-admin -- admin@example.com revoke
```

Admin APIs verify Firebase ID tokens and require the `admin: true` custom
claim plus an allowlisted email. Firestore and Storage rules enforce the same
email restriction. Firestore and Storage client rules are additional
enforcement, not a replacement for server-side validation.

### Deploy Firestore and Storage rules

Select the correct Firebase project before deployment, then deploy the reviewed
rules and indexes:

```powershell
firebase use --add
firebase deploy --only firestore:rules,firestore:indexes,storage
```

Confirm the target project before every deploy. Test rule changes with the
emulators first.

## Application areas

- Public pages: home, service menu, product shop, gallery, about, price list,
  privacy, and terms. Phone, email, Instagram, WhatsApp, and map links are
  available site-wide in the footer instead of a separate contact page. The
  brochure-based price list has separate searchable pages for each salon
  collection; it is a reference menu and does not invent the appointment
  durations needed to enable booking.
- Firebase Authentication: email/password, Google sign-in, and verified Indian
  phone linking.
- Booking: live availability derived from salon hours, service durations,
  staff schedules, lead time, and transactionally-created slot locks.
- Shopping: per-user persisted cart and wishlist; order prices and stock are
  revalidated in a Firestore transaction.
- Reviews: purchase/appointment eligibility checks, owner-scoped image uploads,
  public approved reviews, and admin moderation.
- Admin: custom-claim-gated content editor, image upload, appointments,
  customers, reviews, and overview analytics.
- Admin device data: admins can review and export all site records, or
  synchronize a read-only offline snapshot encrypted in this browser. The
  snapshot requires a separate 12-character local passphrase, expires after
  24 hours, and is cleared on sign-out or account change. Offline authorization
  is limited to the device-data page; other admin pages require an online
  Firebase admin-claim check. Exports include sensitive customer and transaction
  data, so use only a passcode-protected admin device and avoid shared devices.
  Syncing reads every available record and can incur Firestore/Auth read costs.
- Search, WhatsApp contact/booking summaries, sitemap, robots rules, JSON-LD,
  and responsive accessible navigation.

The admin content editor uses validated JSON documents rather than dedicated
field forms. Add real approved copy, service schedules, catalog entries, and
optimized images before launch.

## Checks and tests

```powershell
npm run lint
npm run typecheck
npm test
npm run test:rules
npx playwright install chromium
npm run test:e2e
npm run build
```

`npm run test:rules` requires the Firestore emulator. In PowerShell, set
`$env:FIRESTORE_EMULATOR_HOST = "127.0.0.1:8080"` in the test terminal first.
The end-to-end smoke tests start the local Next.js development server and need
the Playwright Chromium browser installed.

## Production deployment checklist

1. Set `NEXT_PUBLIC_SITE_URL` and all required Firebase web variables in the
   deployment platform's production environment.
2. Set Admin credentials as server-only encrypted secrets, or use the hosting
   provider's supported Application Default Credentials. Never add Admin
   credentials to client bundles.
3. Configure Firebase Authentication providers, OAuth origins, phone quotas,
   authorized domains, Firestore, and Storage.
4. Deploy Firestore rules, Storage rules, and indexes to the intended
   production project. Confirm the admin custom claim works.
5. Add the approved services, staff schedules, prices, stock, business hours,
   verified contact details, and original licensed imagery. Test one booking
   and one order with a controlled account.
6. Review SMS costs, order fulfilment, cancellation/refund handling, backups,
   logging, rate limits, and monitoring.
7. Review privacy, terms, returns, cancellations, data retention, consent, and
   applicable Indian consumer/data-protection obligations with qualified
   counsel before public launch. The included legal pages are starter content,
   not legal advice.

Vercel can deploy this Next.js app from the Git repository. Set environment
variables for each environment in Vercel, connect the production domain, and
verify the generated `/sitemap.xml` and `/robots.txt` using that domain. Avoid
deploying Firebase service-account JSON files or `.env.local`.

## Product assumptions

- The supplied transparent brand logo is used with the Playfair Display
  wordmark and plum accents.
- No stock or hotlinked product imagery is included. Use original or licensed
  imagery uploaded through the admin.
- The initial payment methods are Pay at Salon / Cash on Delivery with WhatsApp
  order summaries. A payment gateway such as Razorpay is not included.
- Business hours default to 10:00 AM–8:00 PM daily until edited in site
  settings. Booking availability still requires active services and staff
  schedules.

## Source prompt status

`SOURCE_PROMPTS_TO_MERGE.md` still contains placeholders for three shared
conversations. Their links did not expose the conversation text in this
environment; the source text is needed before a complete cross-source
requirements merge can be verified.
