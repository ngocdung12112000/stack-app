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
