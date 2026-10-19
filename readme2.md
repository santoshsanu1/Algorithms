# Security and Technical Re-audit

**Audit date:** 20 October 2026  
**Scope:** Source review of the current Django application after the recent fixes, with focus on authentication, public listing/search, map rendering, uploads, inquiry handling, payment completion, expiry, admin workflows, and deployment settings.  
**Environment:** Local checkout; production services are Render, Neon PostgreSQL, Cloudinary, Firebase Phone Auth, and Brevo. No production credentials or services were accessed.  
**Changes made:** This report only. No application code, configuration, or data was changed.

## Summary

The recent security and workflow fixes are present in the source. The most important newly identified issue is a **stored cross-site scripting (XSS) path in the property map**: seller-controlled listing fields are serialized into executable JavaScript and then inserted as popup HTML. I also found missing throttling/account-enumeration protections around authentication and a listings page that materializes all matching properties before pagination. These deserve attention before treating the site as robust under hostile traffic or larger inventory.

## Findings

### P1 — Stored XSS through property map popup data

**Evidence:** `properties/views.py:510–557` builds `map_properties` from property title, locality, formatted price/rate, area, image URL, and detail URL, then serializes it with `json.dumps`. `templates/properties/property_list.html:669–670` inserts that JSON directly into an executable `<script>` using `|safe`. The popup renderer then interpolates values into HTML template strings at `:683–690` and `:773–810`, including the listing title/locality, image URL, and detail URL.

**Impact:** A seller who can create or edit a listing can put HTML/JavaScript payloads in text fields such as the title or locality. A visitor opening the listings/map page can execute attacker-controlled script in the site origin. This can act as the visitor, read page data, or manipulate the UI. The direct script embedding also creates a `</script>` breakout path; escaping only HTML popup values would not address that separate context.

**Recommendation:** Serialize data using Django's `json_script` (or an equivalent serializer that escapes `<`, `>`, and `&`) instead of `|safe` in executable JavaScript. In the map code, build popup elements with DOM APIs and assign seller-controlled values using `textContent`; validate URLs and set them through DOM properties. If Leaflet requires an HTML string, escape every untrusted value for HTML and attribute context, while still using safe JSON transport. Add a regression test using a title containing `</script><img src=x onerror=...>` and markup in locality/image-related values.

### P2 — Login and account recovery lack abuse throttling; recovery reveals account existence

**Evidence:** `accounts/views.py:337–404` handles login without an application-level attempt limit. `accounts/views.py:495–529` returns “No account found with that email address” for unknown password-reset addresses, and `:308–334` similarly distinguishes unknown addresses in verification resend. The phone verification request routes at `:171–241` also have no visible per-IP/user cooldown in this code path.

**Impact:** Attackers can automate password guesses and account discovery. Recovery and resend endpoints can be used to enumerate registered emails. Firebase's own SMS controls may reduce OTP abuse, but application-level throttling and Firebase project restrictions should be verified because unauthenticated users control request volume.

**Recommendation:** Add shared per-IP and per-account throttling for login, password reset, verification resend, and phone verification; use a progressive delay or challenge after repeated failures. Return the same generic response for known and unknown emails on reset/resend. Configure and review Firebase phone-auth quotas, allowed domains, and abuse controls. Keep legitimate UX by showing a generic “if an account exists” confirmation.

### P2 — Property listing page loads and serializes every matching listing before pagination

**Evidence:** In `properties/views.py:510–535`, map data is assembled by iterating over `properties`; this occurs before `Paginator` is applied at `:537–540`. With a text query, `fuzzy_filter_properties` at `:151–189` scores all database candidates in Python and returns a fully materialized list. The map JSON then includes all matching rows, even though the page displays nine results.

**Impact:** Work and response size grow with the entire inventory for every list-page request. Broad/short fuzzy queries can produce a large candidate set and consume CPU/memory. This can slow legitimate visits and make resource exhaustion easier as listings grow.

**Recommendation:** Paginate before serialization and only send map markers for the current page or a separately bounded map viewport. Bound query length and candidate count; move fuzzy matching into an indexed search service or database-backed strategy when inventory grows. Add timing/query-count checks with a realistically large fixture set.

### P2 — User notifications are sent synchronously after commit and can delay user-facing requests

**Evidence:** Inquiry notifications are scheduled with `transaction.on_commit` in `properties/views.py:1490–1498`; payment confirmation email callbacks are scheduled in `properties/payment_service.py:185–190` and `:220–225`. The callbacks call mail delivery directly (`properties/notifications.py:28–49`). `transaction.on_commit` prevents sending for a rolled-back write, but it runs the callback synchronously in the request process.

**Impact:** Slow Brevo/provider responses can hold a web worker and delay the inquiry/payment response. Exceptions are caught, so the core write remains committed, but a failed notification has no durable retry record and can be missed.

**Recommendation:** Persist an outbound-notification job/event in the same database transaction and deliver it through a worker or managed queue with retries and idempotency. Until a queue is available, keep a strict provider timeout and alert/record delivery failures so failed notifications can be retried operationally.

### P3 — Expiry reminder command holds a row lock while making a network mail call

**Evidence:** `properties/management/commands/send_listing_expiry_reminders.py:35–39` starts an atomic transaction and locks a property row; it calls `send_mail` at `:75–82` before writing the sent timestamp at `:87–89`.

**Impact:** A slow mail provider keeps the PostgreSQL transaction and row lock open. If the email is accepted but the database commit fails, a later cron run can send the same reminder again because the marker was not saved.

**Recommendation:** Claim reminders with a short-lived database status/lease, commit that claim, send outside the lock, then mark success or release/retry on failure. Make the claim safe when two Render cron executions overlap. Monitor skipped and failed recipients.

### P3 — Logout changes session state through GET

**Evidence:** `accounts/views.py:407–411` logs the user out without restricting the HTTP method. The route is linked as a normal navigation URL.

**Impact:** A third-party page can cause a logged-in browser to visit the logout URL and end the session (a nuisance/session-disruption issue; it does not grant the attacker access).

**Recommendation:** Make logout POST-only with a CSRF-protected form and update navigation accordingly.

## Areas reviewed with no new blocker found

- Production settings fail fast when PostgreSQL or Cloudinary is not configured; secure cookies, HTTPS redirect, proxy HTTPS handling, and HSTS are enabled in production (`config/settings.py:65–83`, `:132–157`, `:190–225`).
- Payment verification is POST-only, checks transaction ownership/session access, verifies gateway signature and captured amount/order/currency, then applies entitlement and transaction state atomically (`properties/views.py:1837–1908`, `properties/payment_service.py:107–242`). A signed provider webhook would still improve recovery when a browser never returns, but I did not count its absence as a newly confirmed defect.
- Upload validation checks size, file signatures, file counts, total size, and supported formats (`properties/views.py:1649–1687`, `:1758–1796`). Browser/proxy request-body limits should also be configured to reject oversized requests before Django parses them.
- Inquiry writes require login and POST, validate ownership, detect account-level duplicates, impose a per-account hourly limit, and ignore a filled honeypot (`properties/views.py:1443–1503`). There is still no IP-level throttle, so the account controls above should be complemented with edge/IP rate limiting.
- Public seller phone display is consent-gated in both rendered listing pages and the API (`accounts/models.py:79–86`; `properties/views.py:1571–1597`).
- Saved-property and listing status mutations are POST-only; redirects from saved-property toggles are same-host validated (`properties/views.py:1400–1440`).
- `.env` is ignored by Git and is not tracked in this checkout. Its contents were not read.

## Verification performed

- `conda run -n django python manage.py check` — passed: no issues.
- `conda run -n django python manage.py makemigrations --check --dry-run` — passed: no model changes missing migrations.
- `conda run -n django python manage.py test` — passed: 71 tests in 81.9 seconds. Django's system check also passed during test startup. The suite prints expected output from tests that simulate email/network failures and rejected HTTP/CSRF requests; these did not fail the suite.

## Suggested correction order

1. Fix the stored XSS in map data transport and popup rendering; add regression tests.
2. Add authentication/recovery throttling and generic recovery responses.
3. Bound listing/map/fuzzy-search work so request cost does not scale with every database row.
4. Move outbound notifications out of web/cron transactions and provide retry visibility.
5. Make logout POST-only.
