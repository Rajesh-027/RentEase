# RentEase: Product Specification
(Next.js App Router + TypeScript + Supabase + Razorpay + Vercel)

A complete, secure, white-label-able hyperlocal rental + services marketplace with
four roles: tenant, landlord, provider, admin. All code must be complete: no
placeholders, no TODOs, no skipped files.

STACK AND DEPENDENCY ALLOW-LIST (use nothing else)
next (App Router, React Server Components), react, react-dom, typescript (strict),
tailwindcss (design tokens as CSS variables), @supabase/supabase-js,
@supabase/ssr, zod, next-intl, leaflet (client-only, dynamic import, OSM tiles),
razorpay (official SDK, server-only), resend (optional, server-only),
@radix-ui primitives (dialog, dropdown, tabs, tooltip, popover) for accessibility
only, lucide-react, server-only, vitest, @playwright/test (optional e2e).
NOT allowed: UI kits (MUI, Chakra, Bootstrap, shadcn copy-paste templates left
unmodified), chart libraries (build a dependency-free canvas chart component with
an accessible data-table fallback), analytics/tracker SDKs, jQuery, state libraries
(use React state, context, URL state, and server data).
Maps degrade gracefully to a list if Leaflet or tiles fail.

PRODUCT SUMMARY
RentEase brings room discovery, roommate sharing, landlord-tenant management and
everyday living services (cleaning, laundry, food/tiffin, repairs, moving, water
delivery) into one platform for students, bachelors and working professionals in
Tier 2 and Tier 3 Indian cities.
Tagline: "Find Your Space. Share Your Living. Simplify Your Everyday."
This is NOT a cart-and-checkout store. The core engine is:
LISTINGS + ENQUIRIES + QUOTE-BASED BOOKINGS + TRUST.
Zero brokerage on rooms by default.

OUT OF SCOPE (do not build): rent collection, e-rental agreements, third-party KYC
APIs, native apps, WhatsApp Business API (only "Share on WhatsApp" links), saved-
search alerts, video calls.

HARD INVARIANTS (the build's spine; every one must hold)
1. Secrets never in the browser. Only NEXT_PUBLIC_ env vars reach the client.
   SUPABASE_SERVICE_ROLE_KEY, RAZORPAY_KEY_SECRET, RAZORPAY_WEBHOOK_SECRET,
   CRON_SECRET, OWNER_EMAIL and RESEND_API_KEY are read ONLY in server-only
   modules (lib/server/*, each starting with `import "server-only"`) and Route
   Handlers/Server Actions. The service-role client is created only in
   lib/server/supabase-admin.ts. Zero credentials in source. A CI check (script)
   fails the build if any server var name appears in client bundles.
2. RLS on every table, default deny, explicit policies. Users touch only their own
   rows. Public read = published+approved listings and public provider profiles
   only. Admin access via SQL function is_admin(permission) reading admin_users.
   No client may write: roles, verification status, payment status, rating
   aggregates, commission, or booking price after quote acceptance. Those change
   only via SECURITY DEFINER RPCs (with fixed search_path) or the service role.
3. Roles are not self-assignable to admin. Users may self-grant ONLY tenant,
   landlord or provider through RPCs set_initial_role / add_role. Admin is granted
   only by the setup route (first owner) or by a super_admin through an audited RPC.
4. First-owner bootstrap (no "first user wins"): POST /api/setup runs with the
   service role, requires a valid session whose email is VERIFIED
   (email_confirmed_at set) and equals env OWNER_EMAIL, requires that no admin
   exists and platform_settings is empty, then in ONE transaction writes the
   settings row and creates the super_admin. After that it permanently refuses.
   platform_settings has NO client insert policy.
5. Contact privacy. Phone/WhatsApp/email of landlords, providers and roommate
   seekers, and the exact street address of rooms, are NEVER returned by public
   queries, views, server components, or page props/RSC payloads. Exposed only by
   RPC reveal_contact(enquiry_id) after the owner ACCEPTS (for roommates: both
   accept). Every reveal is logged in contact_reveals. Map pins and listing pages
   show an approximate location (coordinates rounded to ~300 m) until reveal.
   Review every server component and generateMetadata/JSON-LD to confirm no
   private field is serialized to the client.
6. Payments verified server-side (only for paid features: booking advance,
   featured listing, provider subscription). Amounts are integer PAISE, currency
   INR. Flow: client calls RPC create_payment_intent (RLS: own only) ->
   POST /api/payments/create re-reads the amount from the DB (NEVER trusts the
   client), creates the Razorpay order with the secret -> Checkout opens with the
   PUBLIC key id (checkout.razorpay.com script loaded with the CSP nonce) ->
   POST /api/payments/verify recomputes HMAC SHA256 of order_id|payment_id,
   compares timing-safe, and only then marks paid via service role ->
   POST /api/payments/webhook verifies the signature against the RAW body
   (use `await req.text()`, never req.json() before verifying), is IDEMPOTENT
   (unique event id in webhook_events, processed in one transaction), and handles
   captured/failed/refunded. Benefits (featured days, subscription days, booking
   advance_paid) are granted only after a verified paid state. Double clicks must
   not create duplicate Razorpay orders (one open order per intent). Cash-on-
   service creates no payment row until the provider marks cash collected AND the
   tenant confirms within a configurable window (else it auto-confirms but is
   flagged for admin review if the tenant disputes).
7. XSS-safe + parameterized. Rely on React escaping. `dangerouslySetInnerHTML` is
   BANNED (ESLint rule) except one JsonLd component that serializes with
   JSON.stringify and replaces "<" with "\u003c". Validate and allow-list every
   user-supplied URL (http/https only; never javascript:). Supabase client/RPC
   only; no string-built SQL. Validate file type (magic bytes where possible),
   size and sanitize filenames on upload.
8. Private documents. Verification docs go to a PRIVATE bucket (verification-docs),
   readable only by the uploader and admins with the verify permission, via
   short-lived signed URLs (60 s) created server-side. Never store a full Aadhaar
   number as text; the UI instructs users to mask it. Docs are deleted after a
   configurable retention period (default 180 days after decision) by the cron job.
9. White-label = data only. Brand name, logo, colors, support contacts, grievance
   officer, cities served, commission %, policy numbers (expiry hours, caps,
   refund rules) and feature toggles come from the singleton platform_settings
   row, read through a cached server function getSettings() (tagged cache,
   revalidated on admin save). Rebranding or adding cities needs zero code changes.
10. Strict CSP with nonces. middleware.ts generates a per-request nonce and sets
    Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-<n>'
    'strict-dynamic' (plus https://checkout.razorpay.com); style-src 'self'
    (Tailwind compiled, no inline style attributes or tags; set dynamic colors
    via CSS variables applied through a nonce'd style element or CSSOM);
    connect-src 'self' + Supabase URL + wss + Razorpay; frame-src Razorpay;
    img-src 'self' data: blob: Supabase storage + OSM tiles; frame-ancestors
    'self'; base-uri 'self'; form-action 'self'; no unsafe-inline, no unsafe-eval
    in production. Theme (light/dark) is applied before paint by a nonce'd inline
    script in the root layout reading a cookie, so there is no flash. Document the
    trade-off that nonces make pages dynamic and how public pages stay fast
    (cache at the data layer with tagged fetch/unstable_cache plus edge caching of
    static assets).
11. Auth and CSRF. Sessions use @supabase/ssr cookies (httpOnly where possible,
    Secure, SameSite=Lax). Middleware refreshes the session. Every mutating Route
    Handler: (a) requires a valid session via supabase.auth.getUser() (never
    trust getSession() on the server), (b) rejects requests whose Origin/Host
    does not match NEXT_PUBLIC_SITE_URL, (c) accepts only POST/PUT/PATCH/DELETE
    with application/json, (d) validates input with zod. Webhook and cron routes
    are exempt from session auth and use their own verification (signature /
    CRON_SECRET bearer, timing-safe). Errors use one JSON shape:
    { error: { code, message, requestId } } and never leak stack traces.
12. Server is the source of truth: route gating via middleware and layout checks
    is for UX; every rule (state transitions, caps, ownership, verification gates)
    is enforced in SQL/RLS/RPC. Client checks are never relied upon.
13. Caching safety. Authenticated pages and any response containing user data are
    dynamic with Cache-Control: private, no-store. Public pages (home, room
    listing, room detail, provider pages, city pages) use ISR/tagged caching with
    revalidateTag fired when a listing is published, paused, edited, or reported
    into auto-hide. Cached public data is fetched with the ANON client only,
    never with a user session or the service role.

ROLES AND CAPABILITIES
Accounts: every user is a tenant; a user may ALSO hold landlord and/or provider
  via add_role (landlord/provider features stay locked until verified). A role
  switcher appears in the navbar. user_roles table holds roles; admin_users holds
  admin tiers: super_admin, moderator, support, each with permissions (verify,
  moderate, reports, settings, users, finance).
Tenant: profile, search rooms, favorites, browse/post roommate posts, enquire,
  request services, accept quotes, pay advance or choose cash, track bookings,
  raise maintenance on a linked tenancy, review after completion, report/block.
Landlord: create/edit/publish/pause rooms, availability, manage enquiries and
  visit slots, invite tenants to a tenancy, handle maintenance, view stats.
Provider: service profile, categories, service areas, weekly availability +
  exceptions, price list, receive requests, send quotes, manage bookings, mark
  complete/cash collected, view earnings and payout ledger.
Admin: users, verification queue, listing moderation, reports, complaints and
  disputes, finance (commission, refunds, payouts), settings, analytics, audit.

CORE FLOWS (explicit state machines enforced in SQL RPCs with SELECT ... FOR UPDATE;
every transition validates the actor, the current state and timing, and writes an
audit/notification row)
A) Room enquiry: sent -> accepted | declined | expired (after enquiry_expiry_hours,
   default 72) -> visit_scheduled -> visited -> closed (tenant may also withdraw).
   Contact and exact address revealed only at accepted. One active enquiry per
   tenant per room. Landlord can propose up to 3 visit slots; tenant picks one.
B) Service booking (quote-based):
   request_posted (category, description, up to 5 photos, preferred time window,
   locality; expires after request_expiry_days, default 7) -> quotes_received (up
   to max_quotes_per_request, default 5; each quote: price in paise, duration
   estimate, proposed slot, validity quote_expiry_hours default 48) ->
   quote_accepted (price LOCKED; all other quotes auto-rejected; accepting twice
   fails) -> advance_paid | cash_selected -> in_progress -> completed -> reviewed.
   Also: rescheduled (max 2, both sides agree), cancelled (refund per policy),
   no_show (provider or tenant), disputed (see F2). Tenant caps: max open requests
   per tenant from settings.
   Recurring bookings (daily tiffin, weekly cleaning): a subscription with pause/
   resume/cancel, generated booking_occurrences (rolling 30 days), per-occurrence
   status (scheduled, done, skipped, missed). Payment: prepaid per cycle via one
   Razorpay payment or cash per occurrence, chosen at acceptance.
C) Roommate matching: a post is either "seeking a room/roommate" or "vacancy in a
   room". Users must be 18+ (confirmed at signup and when posting). Filters: city,
   budget, gender preference (incl. women-only), occupation, lifestyle tags (veg/
   non-veg, smoking, drinking, sleep schedule, pets). Connect via request; contact
   revealed only on mutual accept. Only locality (not address) is shown.
D) Tenancy and maintenance: landlord invites tenant (tenancy_invites) -> tenant
   accepts -> tenancy active -> ended (by either side with date). Maintenance only
   on an active tenancy: raised -> acknowledged -> assigned (optionally to a
   provider, which creates a linked service booking) -> resolved -> confirmed by
   tenant (auto-confirm after 7 days).
E) Verification: user uploads docs -> pending -> approved | rejected (with reason)
   -> optional expired (re-verify after verification_validity_days). Docs by role:
   landlord = ID proof + ownership proof (electricity bill/sale deed/tax receipt);
   provider = ID proof + (shop licence | skill certificate | self-declaration for
   small providers, flagged "basic"). Badge levels: basic, verified. Unverified
   landlords/providers can draft but not publish or quote. If verification is
   revoked/expired, their listings are auto-paused by trigger.
F) Reviews and disputes:
   F1) Only the tenant of a completed booking or visited/tenanted room may review;
   one per booking/tenancy; aggregates recomputed by trigger; provider/landlord may
   reply once; admin may hide abusive reviews (audit logged).
   F2) Dispute: opened by tenant or provider within dispute_window_days after
   completion -> evidence from both sides (text + up to 5 photos) -> admin
   resolves with one of: full refund, partial refund (amount), release to provider,
   reject; resolution triggers /api/payments/refund where money is involved.
G) Listing lifecycle: draft -> pending_review (if moderation_mode = pre_approval)
   or published directly (if post_moderation) -> paused | expired | rejected.
   Rooms auto-expire after listing_validity_days (default 60) unless the landlord
   confirms "still available" (reminder at day 50). Duplicate detection: block
   publishing a room with the same landlord + same locality + same rent + same
   title similarity above a threshold. Per-user daily cap on new listings.
H) Reports and safety: reports on listings/profiles/reviews/messages with reason
   enum; a listing auto-hides pending review when report count >= report_autohide_
   threshold (default 3 distinct reporters); block user = hide each other's
   content and disable messaging both ways (enforced in RLS and RPCs).

MONEY RULES (all configurable in platform_settings; all optional toggles)
- Commission: commission_percent applies to online-paid service bookings only.
  commission_paise = round(price_paise * percent / 100), stored on the booking at
  acceptance time. Provider_earning = price - commission. GST on commission is
  shown on the receipt as a configurable gst_percent (default 18) on the commission
  component; the platform GSTIN is a settings field.
- Advance: advance_percent (default 20, 0 allowed) of the locked price.
- Cancellation/refund policy as data: full refund if cancelled before
  free_cancel_hours of the slot, else refund_percent_late (default 50); provider
  no-show = full refund. Refunds executed ONLY by /api/payments/refund (service
  role, admin or system), recorded in payments with a refund row, never by client.
- Payout ledger: provider_ledger rows (credit on completion, debit on refund or
  payout). Admin marks payouts done with a reference (manual UPI/bank transfer);
  document how to upgrade to Razorpay Route later. No auto-payout in v1.
- Featured listing (rooms/providers) and provider subscription tiers are ONE-TIME
  period purchases (e.g. 7/30/90 days), not recurring mandates. Prices and
  durations come from platform_settings.plans (jsonb, validated by CHECK/RPC).
- Receipts: receipt email template filled server-side, includes booking number,
  amounts, commission, GST, and platform details.

DATABASE (4 SQL files, run in order; all idempotent where possible)
schema.sql: extensions (pgcrypto, pg_trgm, citext, postgis, unaccent); enums for
  every status/role/reason; all tables with UUID PKs, FKs, CHECK constraints
  (amounts >= 0, rating 1-5, date ranges), indexes (GIN trigram on titles/
  localities, GIST on geography, composite city+price+type+status), updated_at
  triggers, soft delete (deleted_at), slugs (unique, generated), all timestamps
  timestamptz (display in Asia/Kolkata).
  Tables: profiles, user_roles, platform_settings, cities, localities,
  verification_requests, rooms, room_images, amenities, room_amenities,
  room_enquiries, visit_slots, tenancy_invites, tenancies, maintenance_requests,
  roommate_posts, roommate_connections, service_categories, provider_profiles,
  provider_services, provider_availability, provider_availability_exceptions,
  service_requests, service_request_photos, quotes, bookings, booking_occurrences,
  payments, payment_intents, webhook_events, provider_ledger, favorites, reviews,
  review_replies, conversations, messages, notifications, notification_prefs,
  reports, blocks, complaints, disputes, dispute_evidence, audit_log (append-only:
  no update/delete policy and REVOKE update/delete), contact_reveals, rate_limits,
  consents, deletion_requests, admin_users.
functions.sql: signup trigger (creates profiles + tenant role; stores consent
  record); RPCs: set_initial_role, add_role, send_enquiry, respond_enquiry,
  propose_visit_slots, reveal_contact, post_service_request, submit_quote,
  accept_quote, cancel_booking, reschedule_booking, mark_cash_collected,
  complete_booking, open_dispute, resolve_dispute, invite_tenant, respond_
  tenancy_invite, end_tenancy, raise_maintenance, update_maintenance,
  submit_review, reply_review, create_payment_intent, report_content, block_user,
  request_data_export (returns JSON of the user's own data), request_account_
  deletion; search_rooms(filters jsonb, page, sort) with pagination and a hard page
  size cap; nearby_providers; check_rate_limit(action, limit, window) used inside
  every write RPC; triggers for rating recalc, notification fan-out, listing
  auto-pause on verification loss, report auto-hide, phone-number masking
  helper for messages; booking-number generator (e.g. RE-YYMM-XXXXXX); batch
  functions expire_stale() and process_deletions(); admin RPCs admin_kpis,
  admin_trends, landlord_stats, provider_earnings, purge_demo_data (super_admin).
  Account deletion: 14-day grace, then anonymize profile, delete private docs and
  messages, keep payment/booking records required for accounting with PII removed.
policies.sql: RLS enabled + explicit policies on EVERY table and storage.objects.
  Buckets: room-images (public read, owner write), avatars (public read, write only
  under avatars/<uid>/), service-photos (participants only, signed URLs),
  verification-docs (private), dispute-evidence (participants + admins).
  image/* only (docs also pdf), per-bucket size caps, path conventions
  <bucket>/<uid>/<uuid>.<ext>. Realtime enabled only for messages and
  notifications, protected by RLS. Public views expose only safe columns.
seed.sql: default platform_settings values (applied by /api/setup, not by client),
  5 demo cities with localities (Kota, Indore, Dehradun, Nagpur, Coimbatore),
  amenities, service categories + subcategories (Cleaning & Housekeeping: room,
  bathroom, kitchen, deep, recurring; Laundry: wash, iron, wash-and-fold, dry
  clean, pickup/delivery; Food: homemade, tiffin, meal subscription, breakfast;
  Repair: plumbing, electrical, appliance, pest control; Other: moving, water
  delivery), demo rooms/providers/roommate posts all flagged is_demo = true,
  using ON CONFLICT DO NOTHING so re-runs are safe.
Also generate types/database.ts (typed Supabase schema) consistent with the SQL.

FEATURES BY AREA
Public (server-rendered, ISR): home (search bar, featured rooms, popular
  localities, service categories, how it works, roommate section, testimonials,
  FAQ, landlord/provider CTAs); rooms listing with filters in the URL query string
  (city, locality, budget range, room type, occupancy, furnishing, amenities,
  availability date, gender preference, verified-only), sort, debounced search,
  pagination, map/list toggle; room detail (gallery, amenities, rules,
  approximate map, verified badge, reviews, enquire, share on WhatsApp, report);
  roommates board; services directory with provider cards, ratings, price-from,
  areas; provider profile with availability and reviews; city landing pages
  (/c/<city>) for SEO.
Auth: email/password, register with role choice and consent checkbox + 18+
  confirmation, forgot/reset, Google OAuth, magic link, email verification
  (required before enquiring, posting, quoting or paying), onboarding page for
  OAuth/magic-link users to choose their first role, optional phone capture.
  Auth callback route exchanges the code for a session.
Dashboards: separate route per role, gated in middleware/layout AND by RLS.
Messaging: per-enquiry/booking conversation thread via Supabase realtime with
  polling fallback; unread counts; phone/email patterns masked with a visible
  warning; no attachments in v1; blocked users cannot send.
Notifications: in-app bell + email via Resend (no-ops without key) for: enquiry
  received/accepted/declined, quote received/accepted, booking confirmed/
  rescheduled/cancelled/completed, maintenance updates, verification result,
  listing expiring, dispute updates. Users can set preferences; emails are
  throttled/deduplicated.
Favorites: rooms, providers, roommate posts.
Admin: dashboard KPIs + dependency-free canvas charts (with accessible data-table
  fallback), user management, verification queue (view signed docs, approve/reject
  with reason), listing moderation, reports, complaints/disputes with resolution
  actions, finance (refunds, payouts, commissions), settings, audit viewer,
  city/locality/category CRUD.
Analytics definitions (document in ARCHITECTURE.md): GMV = sum of locked prices of
  completed bookings; take rate = commission / GMV; enquiry-to-accept rate;
  quote-to-accept rate; time-to-first-quote; listing fill rate.

INDIA / TIER 2-3 REQUIREMENTS
Mobile-first, installable PWA (public/manifest.webmanifest + hand-written
  public/sw.js). SW rules: precache a minimal app shell with a versioned cache
  name (version injected by a build script), runtime-cache public listing images
  and last viewed public listings, NEVER cache authenticated pages, RSC payloads
  for authenticated routes, API or Supabase responses, show /offline when offline,
  in-app "update available" prompt.
  Acceptable on slow 3G: first-load JS <= 100 KB gzipped per public route
  (enforced by a CI script reading the Next build output), server components by
  default, "use client" only where interaction requires it, dynamic import for
  Leaflet, charts, and the Razorpay loader; next/image or explicit width/height
  with lazy loading; client-side compression and EXIF stripping via canvas before
  upload, two sizes generated (400 px thumb and 1280 px full); low-data mode
  toggle (no map tiles, thumbs only, no autoplay/animations), stored in a cookie
  so the server can honor it.
  Hindi + English via next-intl with messages/en/<namespace>.json and
  messages/hi/<namespace>.json and a language switcher (data-driven so more
  languages can be added as JSON only; fallback to English for missing keys; fully
  translated Hindi; hreflang tags).
  INR formatting with Indian digit grouping (Intl en-IN). UPI-first via Razorpay.
  Share on WhatsApp on every listing. Large tap targets (>= 44 px).
DPDP Act and legal: consent checkbox at signup stored in consents (purpose,
  version, timestamp); privacy policy and terms pages written in full, bound to
  brand/grievance settings, with an intermediary disclaimer, prohibited-content
  rules (no discriminatory listings, no brokerage fees, no illegal services) and
  a note that they should be reviewed by a lawyer; data export and account
  deletion request in profile; a grievance officer name/email in settings and the
  footer; no third-party trackers.

UI / PERF / SEO
Design bar: clean, trustworthy, Airbnb/Zomato-level polish, custom design system
  built on Tailwind with tokens in app/globals.css (NOT template-looking). Dark +
  Light (persisted in a cookie, no flash), brand colors from platform_settings
  applied as CSS variables. Reusable components in components/: Navbar, Footer,
  ListingCard, ProviderCard, Modal, ConfirmDialog, Toast, Skeletons, StarRating,
  Stepper, EmptyState, ErrorState, CanvasChart, JsonLd. Use loading.tsx, error.tsx
  and not-found.tsx for every route group. Accessible: labels, visible focus
  rings, aria, keyboard navigation, focus trapping in dialogs (Radix), AA
  contrast, reduced-motion support. Lighthouse 95+ (mobile) on public pages.
  SEO: clean slugs (/rooms/<slug>, /services/<slug>, /c/<city>); generateMetadata
  for title, description, canonical, OG/Twitter and hreflang; JSON-LD (Organization;
  Accommodation/LodgingBusiness for rooms WITHOUT exact address; LocalBusiness +
  Service for providers); app/robots.ts (disallow dashboards, admin, api);
  app/sitemap.ts (published rooms, providers, city pages, with lastModified, split
  into multiple sitemaps if over 50,000 URLs); 404 for unpublished listings; no
  indexing of private pages (noindex metadata).

SECURITY
Security headers in next.config.ts (headers()) and middleware: nonce CSP (see
  invariant 10), X-Content-Type-Options nosniff, X-Frame-Options SAMEORIGIN,
  Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy (geolocation
  self only), HSTS, Cross-Origin-Opener-Policy same-origin-allow-popups (for
  Razorpay/OAuth popups). vercel.json holds crons only.
Rate limiting: Postgres-backed check_rate_limit inside RPCs for enquiries, quotes,
  requests, reports, messages, auth-adjacent actions (limits from settings);
  Route Handlers call the same limiter keyed by user id or hashed IP; honeypot
  field on public forms; per-user daily listing caps; document Vercel/Cloudflare
  edge limits and Supabase Auth rate limits and CAPTCHA option.
Audit log: admin actions, role changes, verification decisions, refunds, payouts,
  setting changes, contact reveals. Concurrency: row locks on every state
  transition; unique partial indexes where required (one accepted quote per
  request, one active enquiry per tenant+room, one review per booking).
Logging: structured JSON logs in route handlers with request IDs; no PII or
  secrets in logs. Dependency hygiene: pinned versions, lockfile committed,
  `npm audit` step documented.

ENV (.env.example, each explained in README)
NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY,
NEXT_PUBLIC_RAZORPAY_KEY_ID, RAZORPAY_KEY_SECRET, RAZORPAY_WEBHOOK_SECRET,
NEXT_PUBLIC_SITE_URL, NEXT_PUBLIC_PLATFORM_NAME, NEXT_PUBLIC_SUPPORT_EMAIL,
NEXT_PUBLIC_PAYMENT_MODE (test|live), OWNER_EMAIL, CRON_SECRET,
RESEND_API_KEY (optional), EMAIL_FROM (optional). lib/env.ts validates all env
vars with zod at startup (server vars only on the server) and fails fast with a
clear message. Switching test/live changes only env values, never code.

SETUP WIZARD (app/setup/page.tsx + /api/setup + /api/setup/upload)
Steps: 1) owner signs up/in with OWNER_EMAIL and verifies email; 2) platform name,
tagline, logo, favicon, brand colors; 3) cities served; 4) support contacts and
grievance officer; 5) commission, advance, cancellation and plan settings and
feature toggles; 6) review and launch. Logo/favicon upload through
/api/setup/upload (service role, owner only, only while setup is open). If
platform_settings is empty, middleware redirects every public route to /setup.
Include maintenance_mode (middleware serves /maintenance except for admins) and a
"not configured" banner.

FOLDER STRUCTURE (use exactly these names; add nothing else except generated files)
Root: package.json, next.config.ts, tsconfig.json, tailwind.config.ts,
  postcss.config.mjs, middleware.ts, vercel.json (crons), .env.example,
  .gitignore, .eslintrc.json (ban dangerouslySetInnerHTML, enforce server-only
  imports), README and docs.
app/ (App Router):
  layout.tsx, globals.css, page.tsx (home), not-found.tsx, error.tsx, loading.tsx,
  robots.ts, sitemap.ts, offline/page.tsx, maintenance/page.tsx, setup/page.tsx
  (public)/rooms/page.tsx, rooms/[slug]/page.tsx, roommates/page.tsx,
    services/page.tsx, services/[slug]/page.tsx, c/[city]/page.tsx,
    privacy/page.tsx, terms/page.tsx
  (auth)/login, register, forgot-password, reset-password, auth/callback/route.ts
  (dashboard)/onboarding, tenant, landlord, provider, messages, profile,
    verification  (each with page.tsx, loading.tsx, error.tsx where relevant)
  (admin)/admin/ page.tsx (overview), users, verifications, listings, reports,
    disputes, finance, cities, categories, analytics, settings, audit
  api/setup/route.ts, api/setup/upload/route.ts, api/payments/create/route.ts,
    api/payments/verify/route.ts, api/payments/webhook/route.ts,
    api/payments/refund/route.ts, api/email/send/route.ts,
    api/cron/expire-stale/route.ts (requires CRON_SECRET)
components/: ui/ (primitives), layout/, listings/, services/, bookings/,
  messaging/, admin/, payments/, charts/CanvasChart.tsx, seo/JsonLd.tsx
lib/: env.ts, utils.ts (formatINR, cn, validators), schemas.ts (zod shared),
  settings.ts (getSettings, tagged cache), i18n.ts, analytics-queries.ts, image.ts,
  supabase/client.ts, supabase/server.ts, supabase/anon.ts,
  server/supabase-admin.ts, server/razorpay.ts, server/email.ts,
  server/request.ts (origin check, error shape, request id, rate limit)
actions/: server actions per domain (enquiries, favorites, reports, auth, bookings,
  quotes, listings, reviews, verification, maintenance, tenancy, messages, profile,
  admin)
messages/: en/<namespace>.json and hi/<namespace>.json. Namespaces: common, auth,
  rooms, roommates, services, bookings, dashboard, admin, setup, legal.
  lib/i18n.ts merges all files in a locale folder.
emails/: enquiry-accepted, quote-received, booking-confirmation, maintenance-update,
  verification-result, receipt (server-rendered, escaped; owned by payments-api)
types/: database.ts
sql/: schema.sql, functions.sql, policies.sql, seed.sql
scripts/: check-bundle-size.mjs, check-secret-leaks.mjs, inject-sw-version.mjs
public/: sw.js, manifest.webmanifest, icons (favicon.svg, 192, 512, maskable)
tests/: rls.test.sql, state-machines.test.sql, api.test.ts (vitest),
  xss.test.md, e2e/ (optional Playwright)

DOCS: README.md (overview, features, stack, install, EVERY env var, where to find
Supabase URL/anon/service-role keys, SQL order, auth + Google OAuth + email
templates, storage, Razorpay test/live + webhook setup, cron setup, local dev,
deploy), setup-guide.md, supabase-setup.md, razorpay-setup.md,
deployment-guide.md, SECURITY.md (threat model, CSP/nonce notes, rate limits,
data retention), ARCHITECTURE.md (naming contract + state diagrams + analytics
definitions + server/client boundary map), TESTING.md, PRIVACY-COMPLIANCE.md
(DPDP checklist), OPERATIONS.md (backups, monitoring, moderation playbook,
incident response, how to purge demo data).

NAMING CONTRACT
ARCHITECTURE.md is written in two parts: Part A (database) by db-architect, Part B
(routes, actions, components, i18n, cookies, test ids) by the orchestrator, then
frozen as git tag contract-v1. It must list every table, column, enum value, RPC
name and its argument names, route path and request/response shape, server action
name, exported function/component name, CSS variable, cookie and localStorage key,
i18n key prefix, and test id used across files. Every later file must use exactly
these names. Conventions: snake_case in SQL and JSON, camelCase for functions/
variables, PascalCase for components, kebab-case for CSS classes, named exports
except where Next requires default (page/layout/route files).

ACCEPTANCE TESTS (implement in /tests AND document in TESTING.md; the code must
pass them)
- A client cannot UPDATE profiles.role, user_roles to admin, payments.status,
  bookings.price/commission, verification status, or any rating aggregate.
- A tenant cannot read another user's enquiries, bookings, messages, payments,
  documents or service photos; a stranger cannot read verification-docs.
- Public queries/views, server-rendered HTML, RSC payloads, metadata and JSON-LD
  never contain phone/email/exact address of landlords, providers or roommate
  seekers; reveal_contact fails before accept and logs after.
- A user cannot register or upgrade themselves to admin; /api/setup rejects an
  unverified email, a non-OWNER_EMAIL, and any second run.
- Tampered Razorpay signature returns 400 and changes nothing; amount tampering
  in the client has no effect; a double click creates one Razorpay order.
- Replaying the same webhook event twice changes state once; a webhook with a bad
  signature returns 400.
- A mutating route called with a wrong Origin or without a session is rejected.
- Two providers' quotes: accepting one rejects the other; accepting twice fails;
  concurrent accepts (two simultaneous calls) yield exactly one winner.
- Expired enquiry/quote/request cannot be acted on; expire_stale is idempotent.
- Unverified landlord cannot publish a room; losing verification pauses listings.
- Review only after a completed booking/tenancy, only once; non-participants fail.
- Blocked user cannot message; 3 distinct reports auto-hide a listing.
- Rate limit trips after the configured count and resets after the window.
- Maintenance request fails without an active tenancy.
- Account deletion request anonymizes data after grace and keeps accounting rows.
- XSS payloads (script tags, img onerror, javascript: URLs) in listing titles,
  descriptions, messages, reviews and profile names render as inert text everywhere.
- Authenticated pages return no-store; public pages never render user-specific data.
- check-secret-leaks.mjs finds no server env var name or key in client bundles;
  check-bundle-size.mjs passes the 100 KB per-route budget.
- Lighthouse (mobile) >= 95 on home, rooms, room detail; no CSP violations in the
  console on any page.

GOAL
Clone -> create Supabase project -> run 4 SQL files -> set env -> open /setup as
OWNER_EMAIL -> deploy to Vercel in ~15 min; rebrand and add cities with ZERO code
changes.