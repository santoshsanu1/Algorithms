# Gaya Bhumi Code and Product Review

**Reviewed:** 19 October 2026  
**Environment checked:** Conda `django`, Python 3.11.13, Django 4.2.30  
**Review type:** Source review plus existing automated tests. No application code was changed.

## Executive summary

The repository has a useful foundation: account roles, property listings, inquiry tracking, configurable pricing, local development payment simulation, and a working Django test suite. The 32 existing tests pass on the repository's pinned Django version, and `manage.py check` passes.

Before a public launch, I recommend addressing the stored cross-site scripting path in the map, making payment completion atomic and recoverable, and enforcing upload type/size/privacy rules. Other important work includes retry/refund handling for abandoned checkouts, verified-phone integrity, and clearer moderation/expiry behavior. The current tests mostly cover happy paths; they do not exercise the live payment, Firebase, Cloudinary, or full media-upload integrations.

## Findings by priority

### P1 — Stored XSS through property content on the search map

`properties/views.py` serializes user-controlled property title/locality fields with `json.dumps`, and `templates/properties/property_list.html` inserts the JSON directly into an executable script with `|safe` (view around lines 318–364; template line 666). A seller can put script-closing markup in a listing title or custom locality. The browser can treat it as HTML/script when loading the search page. The Leaflet popup also interpolates title/locality into an HTML string around lines 775–786, which can execute markup when a popup is rendered.

**Recommendation:** Pass the Python data to Django's `json_script` filter and parse its text in JavaScript. That prevents breaking out of the script element. Also render popup values as text using DOM APIs (`textContent`/text nodes), or apply context-appropriate HTML escaping before Leaflet receives the popup. Add a regression test with a title containing `</script>` and markup, plus a browser-level check that it renders as text.

### P1 — Payment status and entitlement can get out of sync

`PaymentService.complete_transaction` marks and saves a transaction as `completed` before it updates the seller or property entitlement (`properties/payment_service.py`, lines 157–179). Those writes are not in one database transaction. If the process fails after the transaction save, the payment row says completed but the account/listing may remain unpaid. A repeated callback then exits early in `payment_verify_view` because it sees `completed` (lines 1328–1335), so it may not repair the entitlement. Concurrent duplicate callbacks can also race through the completion path.

**Recommendation:** Lock and re-read the payment row, apply the entitlement and transaction status in one `transaction.atomic()` block, make completion idempotent, and send email after commit. Add tests for duplicate callbacks, injected failures between writes, wrong user/order, and invalid signatures. Consider a signed Razorpay webhook as a reconciliation path for users who close the checkout page.

### P1 — Uploads have count limits but no visible type or size enforcement

The upload validator only checks the number of gallery images, videos, and documents (`properties/views.py`, lines 1239–1264). Gallery, video, and document records use regular file fields (`properties/models.py`, lines 462–465 and 537–562), and the upload loops save them directly (`properties/views.py`, around lines 925–931 and 999–1007). The field labels promise photos, videos, and PDF/documents, but the server-side path shown here does not validate those types or cap individual/total size.

**Recommendation:** Enforce size and content type server-side (inspect file signatures, not only extensions), restrict accepted formats, and isolate user media from the application origin or force safe download headers for documents. Add tests for invalid and oversized files. This also controls storage and bandwidth costs.

### P1 — Production can silently store uploads on ephemeral local disk

The settings comments say Cloudinary is required on Render because local disk is wiped, but production only checks that `USE_CLOUDINARY` is not false. If `CLOUDINARY_URL` is missing, `USE_CLOUDINARY_STORAGE` becomes false and storage silently falls back to `FileSystemStorage` (`config/settings.py`, lines 189–212). The production configuration checks later in the file do not require Cloudinary.

**Recommendation:** Fail fast in production if persistent media storage is not configured, or explicitly configure a persistent disk/object store. Add a production-settings test for missing and valid media storage configuration.

### P2 — Free listing quota can be consumed before payment succeeds

When a seller has a free category credit but chooses a paid listing plan, the code decrements the free quota as soon as the `payment_pending` property is saved (`properties/views.py`, around lines 913–921). The credit is not restored if checkout is abandoned or order creation fails later (lines 951–968). There is no visible payment retry/release workflow for a pending listing without a transaction.

**Recommendation:** Reserve the credit and consume it only after confirmed payment, or store a reservation that is released on cancellation/expiry. Give sellers a retry/cancel flow for pending listings and reconcile stale orders.

### P2 — Profile edits can retain a verified-phone flag after changing the number

`UserProfileForm` allows changing `phone_number` (`accounts/forms.py`, line 107), and `profile_view` saves it directly (`accounts/views.py`, lines 386–398). This path does not clear `is_phone_verified`. Even if this flag is not currently used to set the per-listing review badge, the account can continue to appear verified in admin and future features may trust stale verification.

**Recommendation:** Make phone edits require OTP, or clear the flag whenever the normalized number changes. Test both unchanged and changed phone submissions.

### P2 — Saved-property state changes through GET

`toggle_save_property` creates/deletes a saved-property record without requiring POST (`properties/views.py`, lines 1052–1071). A cross-site link can trigger this state change for a logged-in user.

**Recommendation:** Use a CSRF-protected POST form and return to a validated same-site destination. Update the current GET-based test.

### P2 — Anonymous inquiry endpoint has no abuse controls

Anyone can submit an inquiry, and successful submissions trigger seller notification (`properties/views.py`, lines 1074–1099). I did not find rate limiting, duplicate detection, a honeypot, or other spam controls. A bot can create many leads and generate outbound email.

**Recommendation:** Add per-IP/account rate limits, duplicate throttling, a honeypot or CAPTCHA after suspicious activity, and an admin spam flag. Queue notification delivery so email-provider delays do not hold the web request.

### P2 — Search map serializes every result before applying pagination

The search view loops over the full filtered queryset to build map markers at lines 318–343, then paginates the cards at lines 345–347. Large result sets therefore still require loading every matching property and building a large JSON payload on each page.

**Recommendation:** Bound map data to the current page or create a map-specific endpoint with viewport bounds and a hard result cap. Add indexes based on actual query patterns and measure query count/response size with a larger fixture.

### P2 — Sensitive contact data is intentionally public in several places

The anonymous search/home templates show seller WhatsApp/phone links, and `/api/properties/` also returns `seller_phone` (`properties/views.py`, lines 1152–1199; `templates/properties/property_list.html`, around lines 601–608). This is not necessarily a bug—the public listing cards appear designed for direct contact—but it enables scraping and may surprise sellers.

**Recommendation:** Make public contact visibility an explicit seller consent setting. Consider inquiry relay, signed/limited contact endpoints, and scraping controls. Keep the API and web UI aligned with the chosen policy.

### P3 — Auto-calculated kattha rate can become stale after edits

`Property.save()` calculates `rate_per_kattha` only when the current value is falsy (`properties/models.py`, around lines 340–343). Since the form includes this field and existing properties already have a calculated value, changing price, area, or area unit can leave the old rate in place.

**Recommendation:** Distinguish seller-entered rates from calculated rates, or recalculate when price/area/unit changes unless the seller explicitly chose an override. Add an edit regression test.

### P3 — Admin settings that look active do not control runtime behavior

`auto_approve_property` is present in settings/admin forms, but code references show no listing workflow reads it. Likewise `WebsiteSetting.standard_listing_fee`, `premium_listing_fee`, and `featured_listing_fee` are editable, while listing checkout reads `ListingCharge` and `ListingPlanChoice`. Admins can change displayed settings that do not affect actual moderation or checkout prices.

**Recommendation:** Wire each setting into the corresponding workflow and test that it changes behavior, or remove/label obsolete controls so operators know which pricing source is authoritative.

### P3 — Listing expiration is stored but not enforced

`Property.expires_at` exists (`properties/models.py`, line 252), but I found no runtime use of the field outside its declaration/migration. If paid plans are meant to expire, listings currently have no visible expiry job or filtering rule.

**Recommendation:** Decide whether plan duration is part of the product. If so, set expiry on successful payment, exclude expired listings from public queries, notify sellers before expiry, and provide renewal/relisting.

### P3 — Mobile API claims pagination but returns only the first 50

The API docstring says pagination is supported, but the view slices `properties[:50]` and returns no next-page cursor or page parameter (`properties/views.py`, lines 1152–1199).

**Recommendation:** Add explicit page/cursor pagination, stable ordering, validation, and API versioning before a mobile client depends on it.

## Product and feature opportunities

### Trust and safety

- Listing report/flag flow, moderation queue, and audit history for admin decisions.
- Seller verification workflow and clearer, evidence-backed listing review badges.
- Document access controls, retention policy, and explicit seller consent before exposing phone numbers.
- Buyer guidance checklist for registry/title verification, site visits, and safe payment practices.

### Buyer experience

- Saved searches with WhatsApp/email alerts for new matching properties and price changes.
- Side-by-side property comparison, map-drawn search area, and richer locality/area filters.
- Price history and comparable listings by locality, property type, and land unit.
- Favorites and inquiry tracking with notifications when a seller responds.

### Seller tools

- Listing views-to-inquiries conversion analytics and lead reminders/notes.
- Listing quality checklist, draft/unpublished state, and preview before publishing.
- Paid-plan renewal, receipts, refunds, and clear pending/failed-payment recovery.
- Bulk photo management and automated image resizing/compression.

### Platform and operations

- Versioned, documented API with rate limits, pagination, privacy rules, and stable error responses.
- Background jobs for email, stale-payment reconciliation, listing expiry, and cleanup of orphaned media.
- Account export/deletion and privacy controls for phone, email, and inquiry history.
- Accessibility audit, Hindi localization, and SEO metadata/structured data for property pages.

## Automated test status and what it does not prove

- `conda run -n django python manage.py check`: passed.
- `conda run -n django python manage.py test --verbosity 1`: 32 tests passed in 39.2 seconds using Django 4.2.30.
- Tests cover local registration/verification, login redirect safety, role switching, search/detail/inquiry, seller/buyer dashboards, free listing creation, pricing/admin settings, maintenance mode, map context, and notification behavior.
- Dedicated tests are still needed for XSS payload handling, unauthorized edit/delete/follow-up attempts, OTP endpoints, password-reset completion/expiry, paid checkout callbacks/failure recovery, upload type/size limits, private/shareable access, and stale rate/expiry behavior.
- Live Razorpay, Firebase, email-provider, and Cloudinary behavior was not exercised. A passing local suite does not validate those external integrations.

## Suggested remediation order

1. Fix and regression-test the map XSS path.
2. Make payment completion atomic/idempotent and repairable by webhook/reconciliation.
3. Add strict upload validation and require persistent production media storage.
4. Fix payment-pending quota reservation and seller recovery flows.
5. Protect phone verification state and move saved-property mutation to POST.
6. Add inquiry abuse controls and bounded map/API result handling.
7. Resolve stale admin controls, rate calculation, and listing-expiry semantics.
8. Choose the next buyer/seller features based on product goals and usage data.
