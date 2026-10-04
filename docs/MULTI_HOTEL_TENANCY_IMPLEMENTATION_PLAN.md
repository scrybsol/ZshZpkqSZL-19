# Multi-hotel tenancy implementation plan

## Purpose and boundary

This roadmap finishes the remaining work needed for one shared hospitality application to serve independent hotels on their own domains without exposing another hotel’s branding, operational data, loyalty activity, accounting records, or private uploads.

The existing `books_organizations` record remains the canonical hotel/tenant ID. The existing organization-scoped loyalty and Books models are therefore retained; this work connects every request, page, write, and access policy to that organization ID.

This plan intentionally covers only unfinished multi-hotel work. It does not redesign the already organization-scoped loyalty or Books ledgers.

## Delivery rules

- A hostname is a routing input, not proof of permission. The server must normalize and resolve the request hostname against a database allow-list.
- `organization_id` must be database-authoritative. Browser parameters and client-side filters are convenience only and may not grant access.
- Every hotel-owned record must belong to exactly one organization. Ambiguous ownership is a migration failure to resolve, not a reason to keep a null tenant ID.
- Memberships, not the global `user_profiles.role`, determine staff access within an individual hotel.
- Payment webhooks and provider callbacks derive their hotel from the persisted booking/order/event, not from a host header or request body.
- The rollout starts with one configured Sheraton tenant and one isolated non-production test tenant before any second production hotel is onboarded.

## Stage 0 — Verify the deployed baseline before applying changes

**Why:** Repository SQL is not proof that the same migrations, functions, policies, grants, and storage configuration are active in the live Supabase project.

1. Export the live schema, RLS policies, grants, functions, triggers, and storage policies.
2. Compare that export with the checked-in migrations and record the actual Sheraton `books_organizations.id`.
3. Inventory all existing rows that should belong to Sheraton, especially records whose `organization_id` is nullable or absent.
4. Test the current app with a staff and a guest account. Preserve a rollback/export point before changing RLS.

**Exit condition:** The live tenant ID, active policy names, and data-backfill counts are known. The migration can use the real Sheraton organization ID in its explicit configuration block.

## Stage 1 — Establish a database-authoritative tenant registry

**What changes:** Add a one-to-one hotel configuration record for each `books_organizations` row and a separate, unique hostname mapping. Add a security-definer hostname resolver that only returns public branding fields for enabled hotels.

1. Add `hotel_tenant_settings` keyed by `organization_id` for public name, logo, brand colors, timezone, currency, and enabled state.
2. Add `hotel_tenant_domains` keyed by normalized hostname, with a single canonical domain per hotel and optional aliases.
3. Add `resolve_hotel_tenant(target_hostname)` to normalize the hostname and return only the matched active hotel’s public configuration.
4. Add a safe configuration/backfill section that creates Sheraton’s settings and domain after its real organization ID is supplied.
5. Restrict direct tenant-settings/domain writes to organization owners/admins; public callers may use only the resolver.
6. Keep the migration idempotent and fail loudly when tenant-owned legacy rows cannot be assigned unambiguously.

**Files:** New Supabase migration and a centralized, paste-ready SQL script in `supabase/`.

**Exit condition:** Calling the resolver with each configured hostname returns only that hotel’s public identity; unknown or disabled hosts return no tenant.

## Stage 2 — Activate tenant resolution in the running application

**What changes:** Turn the existing inactive provider and server helpers into a real request path.

1. Register `GET /api/hotel-tenant`, `GET /api/hotel-booking-data`, and `GET /api/hotel-menu-items` in `server/index.ts`.
2. Mount `HotelTenantProvider` above the React application so the active hotel identity is available on every route.
3. Make the provider handle unavailable/unknown domains explicitly and reset brand CSS variables when the tenant changes or resolution fails.
4. Use tenant name and logo in the header and homepage. Keep generic fallback copy only while the tenant is loading; do not silently present Sheraton branding on an unknown domain.
5. Normalize trusted forwarded host handling for supported deployment adapters before production custom-domain rollout.

**Exit condition:** The homepage, header, and tenant API resolve from the current configured hostname, rather than static Sheraton constants.

## Stage 3 — Scope public hotel experiences to the resolved tenant

**What changes:** Replace direct global browser reads with tenant-resolved server data and require the same organization in any business transaction.

1. Booking page: fetch booking settings, offers, and published rooms from `/api/hotel-booking-data`; pass the resolved hotel identity through availability and checkout flows. Add `organization_id` to booking settings/offers and scope their uniqueness/order by hotel.
2. Menu page: fetch published items from `/api/hotel-menu-items`; tenant-key carts in local storage and ensure order recovery only resumes an order for the current hotel.
3. Events: add tenant-scoped public event reads, facilities, event plans, and booking/payment operations. The domain determines the hotel; a submitted organization ID is never authoritative.
4. Replace static tenant names in public copy with the resolved hotel name where branding is customer-facing.

**Exit condition:** Visiting Hotel A’s domain cannot list Hotel B’s published rooms, offers, menu items, or public events. Switching domains discards an incompatible pending cart or checkout state.

## Stage 4 — Complete required tenant ownership and tenant-aware authorization

**What changes:** Make all remaining hospitality records traceable to one hotel and remove policies that grant cross-hotel access merely because a user is authenticated.

1. Backfill and then require `organization_id` for hotel-owned rows including menu records/carts, events and supporting event records, tasks, complaints, notifications, and attachment metadata.
2. Add tenant ownership to currently global booking settings, booking offers, event facilities, event plans, carts, and their dependent data. Use parent-record checks for child rows where copying the ID would create unnecessary duplication.
3. Replace broad tasks/complaints/profile role policies with membership-based RLS. A manager/provider at Hotel A receives no access to Hotel B unless there is a membership for Hotel B.
4. Reject tenant-changing updates and cross-organization relationships with foreign keys, triggers, or constraints.
5. Update authenticated management screens so writes obtain their tenant from current membership/domain context and never default to a user’s first organization.

**Exit condition:** Direct Supabase API requests cannot read, insert, update, or attach Hotel B data from a Hotel A account. Every persisted hotel business record has a known tenant or an owning parent that has one.

## Stage 5 — Make payments, accounting, loyalty, and notifications tenant-bound end-to-end

**What changes:** Preserve each financial transaction’s tenant identity from creation through reconciliation and returns.

1. On booking/menu/event checkout, resolve the tenant server-side and verify that selected room, menu item, event, cart, and existing order/booking all match it.
2. Scope availability RPCs, order recovery, cancellation, retry, verification, and ticket issuance by the persisted organization.
3. Store/reuse the canonical tenant host for hosted-payment return URLs. Never build a return URL from arbitrary request headers or body fields.
4. Keep webhook handling host-independent: derive the organization from the persisted transaction association and provider verification.
5. Verify that Books postings and loyalty awards use the same organization as their booking/order/event source. Restrict profile reward views to the active hotel unless cross-hotel discovery is an explicit product feature.
6. Add organization context to notifications and delivery/audit records so staff communications do not cross hotel boundaries.

**Exit condition:** A payment identifier, booking token, idempotency key, or transaction reference from Hotel A cannot be reused or recovered through Hotel B’s domain.

## Stage 6 — Isolate private uploads and document access

**What changes:** Replace public/unscoped attachment ownership with hotel-bound storage and metadata.

1. Store each private file under a prefix such as `hotels/{organization_id}/...` and record the organization on its attachment metadata.
2. Change upload requests to resolve the tenant on the server; do not accept a browser-supplied organization ID as authority.
3. Use private buckets and signed URLs for complaints, tasks, guest details, receipts, and other restricted documents. Keep only deliberately public tenant assets (for example an approved logo) public.
4. Add storage/object policies and attachment-link checks that require membership in the file’s hotel, with guest-specific access where appropriate.
5. Backfill/move existing files after an export and checksum verification. Do not remove the original storage object until the new object and metadata are validated.

**Exit condition:** A Hotel A user cannot list, retrieve, or attach a Hotel B private object by changing a file path, URL, attachment ID, task ID, or complaint ID.

## Stage 7 — Prove isolation and prepare onboarding

**What changes:** Add automated coverage and a controlled two-hotel deployment rehearsal.

1. Create a non-production Hotel A and Hotel B, each with a distinct hostname, manager, provider, guest, rooms, offers, menu items, event, task, complaint, loyalty program, and sample file.
2. Add resolver, route-registration, and cross-tenant integration tests. Cover unknown hosts, normalized hosts, reverse-proxy forwarding, failed resolver calls, and cache behavior.
3. Test direct Supabase access under both accounts, not only what the UI hides.
4. Run booking/menu/event checkout success, cancellation, retry, verification, webhook, Books posting, loyalty awarding, and payment-return flows for both hotels.
5. Verify all supported deployments preserve the configured hostname into Express and that each custom domain has working DNS/TLS before enabling the hotel.
6. Create an operational onboarding checklist: create organization, configuration, canonical domain/aliases, owner membership, hotel-specific roles, default accounting/loyalty settings, storage prefix, validation data, and acceptance test.

**Exit condition:** Automated tests, type checking, and build pass; two test hotels are proven mutually inaccessible at the UI, API, database, storage, and payment levels.

## Implementation order

1. Stage 0 is a release gate and must happen immediately before database rollout.
2. Stages 1 and 2 establish the active tenant context and can be implemented first in the repository.
3. Stage 3 follows before new public hotel features are built.
4. Stage 4 is the security boundary and must finish before onboarding a second real hotel.
5. Stage 5 follows once domain/record associations are reliable; implement it before enabling multi-hotel payments.
6. Stage 6 must complete before private hotel documents are stored for multiple hotels.
7. Stage 7 is the final admission gate for a second production hotel.
