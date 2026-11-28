# F02 — User Profile & Address Book

| Field | Value |
|-------|-------|
| Phase | 1 |
| Service | `identity` |
| Stack | EF Core + MySQL |
| Frontend | `/account`, `/account/addresses`, checkout address step |
| Depends on | F01 |

## 1. User stories

| ID | Story |
|----|-------|
| US-02.1 | As a customer, I can view and edit my name, phone and avatar |
| US-02.2 | As a customer, I can add, edit and delete shipping addresses |
| US-02.3 | As a customer, I can set a default address that checkout pre-selects |
| US-02.4 | As a customer, I can request account deletion (GDPR-style soft delete + anonymisation) |

## 2. Business rules

- BR-01: Max 10 addresses per user.
- BR-02: Exactly one default address if the user has ≥ 1 address. The first address becomes the default automatically. Deleting the default promotes the most recently updated remaining address.
- BR-03: Phone is validated as E.164 (`libphonenumber-csharp`).
- BR-04: Address fields: recipientName (required, ≤ 100), phone (required), line1 (required, ≤ 200), line2, ward, district, city (required), countryCode (ISO-2, required), postalCode (optional by country).
- BR-05: Orders store a **snapshot** of the address, so editing or deleting an address never changes past orders.
- BR-06: Account deletion: status → `Deleted`, email → `deleted+{id}@invalid`, name/phone wiped, addresses deleted, tokens revoked. Publishes `identity.user-deleted.v1` so other services anonymise their data (reviews show "Deleted user").

## 3. API

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/users/me` | Profile |
| PATCH | `/api/v1/users/me` | `{fullName?, phone?, avatarAssetId?}` |
| DELETE | `/api/v1/users/me` | `{password}` confirmation → 202 |
| GET | `/api/v1/addresses` | List (default first) |
| POST | `/api/v1/addresses` | Create → 201 |
| PUT | `/api/v1/addresses/{id}` | Replace |
| DELETE | `/api/v1/addresses/{id}` | 204 |
| POST | `/api/v1/addresses/{id}/default` | Set default |

Errors: `ADDRESS_LIMIT_REACHED` (409), `ADDRESS_NOT_FOUND` (404, also returned for another user's address).

## 4. Frontend

- `/account` overview: profile card with an edit dialog, plus quick links.
- `/account/addresses`: grid of address cards (Badge "Default"), with Add/Edit in a `Dialog` using the shared `AddressForm` from `packages/ui`.
- The checkout address step reuses `AddressForm` and the radio list.
- Optional: country/city select data from a static JSON (Vietnam provinces as an example dataset).

## 5. Acceptance criteria

- AC-1: The first address created is the default automatically.
- AC-2: Setting address B as default unsets A in the same transaction.
- AC-3: An 11th address returns 409.
- AC-4: User X cannot read, update or delete user Y's address (404).
- AC-5: After account deletion, login fails and reviews display "Deleted user".

## 6. Tasks

1. `Address` entity + EF config, unique filtered constraint logic (handled in the domain, since MySQL lacks partial indexes).
2. Handlers + validators. Phone validation.
3. Account deletion command + event.
4. UI pages + `AddressForm` component + tests.

## 7. Learning focus

Aggregate invariants (single default), transactional updates, snapshot vs reference data, privacy-by-design.

## 8. As built (backend, 2026-11-28)

Implemented in `services/identity` (EF migration `AddAddressesAndAvatar`, additive only). Differences and decisions:

- **Paths.** Canonical paths are the ones in section 3 (`/api/v1/addresses`, matching docs/05 and the new gateway route `identity-addresses`). `/api/v1/users/me/addresses` is a hidden alias (same handlers, not in OpenAPI); it is already covered by the `identity-users` gateway route.
- **Error codes** carry the service prefix required by docs/05 section 3: `IDENTITY_ADDRESS_LIMIT_REACHED` (409), `IDENTITY_ADDRESS_NOT_FOUND` (404, also for another user's address). Validation failures are `IDENTITY_VALIDATION_FAILED` (400) with an `errors` map keyed by camelCase field (`recipientName`, `phone`, `line1`, `city`, `countryCode`, `postalCode`, `fullName`). A wrong password on `DELETE /users/me` is `IDENTITY_INVALID_CURRENT_PASSWORD` (400, not 401, so the BFF does not try to refresh).
- **Single default (BR-02).** Rule lives in the `AddressBook` aggregate (Domain). Race safety: every address command and account deletion runs in one transaction that first executes `SELECT ... FOR UPDATE` on the owner's `users` row, so concurrent writers of the same user queue; max-10 and exactly-one-default therefore hold under concurrency (covered by an integration test with 14 parallel creates). No unique generated column is used because switching the default would then depend on EF update ordering.
- **Responses.** `POST /addresses` 201 + body + `Location`; `PUT` 200 + body; `DELETE` 204; `POST /addresses/{id}/default` 200 + the address. `GET /addresses` returns a plain array, default first then most recently updated.
- **Phone (BR-03).** Must start with `+` and be valid per libphonenumber (`libphonenumber-csharp`); stored canonical (`+84901234567`). Country code is any two letters, stored upper-case. `postalCode` is optional for every country (max 20).
- **PATCH /users/me.** Null/omitted = unchanged. `phone: ""` clears the phone; `avatarAssetId: "00000000-0000-0000-0000-000000000000"` clears the avatar. At least one field is required. `avatarAssetId` is only stored (`users.avatar_asset_id`); it is not validated until the media service exists (Phase 4). `GET /users/me` only gained `avatarAssetId` (additive).
- **DELETE /users/me** (`{password}`) returns 202. In one transaction: status `Deleted`, email `deleted+{id}@invalid`, name/phone/avatar wiped, password hash replaced by an unusable value, addresses deleted, all refresh tokens revoked, outstanding email tokens invalidated, outbox row `identity.user-deleted.v1` (schema in `contracts/events`, payload `{userId, deletedAt}`, no PII). The presented access token is then blacklisted by `jti` (BR-09 as for lock). Access tokens of other devices stay cryptographically valid for at most 15 minutes but every endpoint answers 404 for a deleted user and refresh/login fail. The email address becomes free for a new registration.
- **Not in this change.** The review service reaction to `identity.user-deleted.v1` (AC-5 "Deleted user" display) arrives with F-reviews in Phase 4; the Kafka relay for the outbox arrives in Phase 2, so the row only accumulates for now. Order address snapshots (BR-05) belong to ordering.
- **Tests.** `F02_AC*` / `F02_BR*` / `F02_US*` in `Identity.Tests` (domain, validators, handlers) and `Identity.IntegrationTests/ProfileAddressTests.cs` (Testcontainers MySQL + Redis).
