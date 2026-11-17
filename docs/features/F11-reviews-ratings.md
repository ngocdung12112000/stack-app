# F11 — Reviews & Ratings

| Field | Value |
|-------|-------|
| Phase | 4 |
| Service | `review` |
| Stack | MongoDB (documents + aggregation pipeline), Kafka, Redis (helpful-vote rate limit) |
| Frontend | PDP reviews tab, "Write a review" in account orders, admin moderation queue |
| Depends on | F01, F07 (delivered orders), F03, F04, F14 (review images) |

## 1. User stories

| ID | Story |
|----|-------|
| US-11.1 | As a customer who received a product, I can rate it 1–5 stars with a title, text and up to 5 photos |
| US-11.2 | As a shopper, I can see the average rating, a star histogram and reviews sorted by newest/most helpful, and filter by star or "with photos" |
| US-11.3 | As a shopper, I can mark a review as helpful (once) |
| US-11.4 | As a customer, I can edit my review within 30 days (it goes back to moderation) |
| US-11.5 | As support, I can approve or reject pending reviews, with a reason |

## 2. Business rules

- BR-01: Only a **verified purchase** can be reviewed. The review service keeps a `purchases` collection `{userId, productId, orderId, deliveredAt}` filled from `ordering.order-delivered` events, so it never calls ordering synchronously.
- BR-02: One review per (userId, productId) (unique index).
- BR-03: Requires a confirmed email (claim `email_verified`).
- BR-04: New reviews are `Pending`. Auto-approve is allowed if rating ≥ 3 and the text passes a simple bad-word filter (configurable). Otherwise it goes to manual moderation.
- BR-05: Only `Approved` reviews are public and count in the rating.
- BR-06: On approve/reject/edit/delete → recompute the summary with an aggregation pipeline:
  ```js
  [{ $match: { productId, status: "Approved" } },
   { $group: { _id: "$rating", n: { $sum: 1 } } }]
  ```
  → build `{avg, count, histogram}` → upsert `product_rating_summary` → outbox `review.rating-updated`.
- BR-07: Helpful votes: 1 per user per review (`helpfulVoters` array with `$addToSet`; the count is maintained with `$inc` only if added). Rate limited to 30 votes/hour/user in Redis.
- BR-08: Text 10–2000 characters, title ≤ 120. Strip HTML.
- BR-09: When a user is deleted, `userDisplayName` → "Deleted user".

## 3. API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/reviews?productId=&rating=&withPhotos=&sort=newest|helpful&cursor=` | anon | Approved reviews |
| GET | `/api/v1/reviews/summary/{productId}` | anon | `{avg, count, histogram}` |
| GET | `/api/v1/reviews/eligibility?productId=` | user | `{canReview, reason?, existingReviewId?}` |
| POST | `/api/v1/reviews` | user | `{productId, sku?, rating, title, body, imageAssetIds[]}` |
| PUT | `/api/v1/reviews/{id}` | owner | Edit (≤ 30 days) |
| DELETE | `/api/v1/reviews/{id}` | owner / Support | |
| POST | `/api/v1/reviews/{id}/helpful` | user | |
| GET | `/api/v1/me/reviews` | user | My reviews |
| GET | `/api/v1/admin/reviews?status=Pending` | `reviews:moderate` | Queue |
| POST | `/api/v1/admin/reviews/{id}/approve` / `reject` | | `{reason}` |

Errors: `REVIEW_NOT_ELIGIBLE`, `REVIEW_ALREADY_EXISTS`, `REVIEW_EDIT_WINDOW_CLOSED`, `REVIEW_EMAIL_NOT_CONFIRMED`.

## 4. Events

- Consumes: `ordering.order-delivered` (populate purchases), `identity.user-deleted`, `media.image-processed` (image URLs).
- Publishes: `review.review-submitted.v1` (→ notification to support), `review.review-approved.v1`, `review.rating-updated.v1` (→ catalog, search).

## 5. Frontend

- PDP Reviews tab: a summary block (big average, `RatingStars`, histogram bars that filter on click), filter chips, a sort select, the review list (avatar initials, stars, "Verified purchase" badge, photos in a lightbox `Dialog`, a "Helpful (12)" button), and a "Load more" cursor.
- Account → Orders → Delivered item → "Write a review" `Dialog` with an interactive star input, form and image uploader (F14).
- Admin moderation: a DataTable of pending reviews with an expandable row, and Approve/Reject buttons (reject asks for a reason).

## 6. Acceptance criteria

- AC-1: A user without a delivered order for the product gets `REVIEW_NOT_ELIGIBLE`.
- AC-2: After an approved 5★ and 3★ review, the PDP shows avg 4.0, count 2, and the search document `ratingAvg` = 4.0 within 30s.
- AC-3: A second review by the same user → 409.
- AC-4: Helpful voting twice increments only once.
- AC-5: A rejected review is not visible publicly but is visible to its author with the reason.

## 7. Tasks

1. Mongo collections + indexes. Purchases projection consumer.
2. Domain + handlers + moderation workflow.
3. Rating aggregation + outbox.
4. Helpful vote with atomic update + Redis rate limit.
5. Frontend PDP tab, write dialog, moderation UI.
6. Tests (aggregation correctness with Testcontainers Mongo).

## 8. Learning focus

MongoDB aggregation, atomic update operators (`$addToSet`, `$inc`), local projections instead of sync calls, moderation workflows, cross-service read-model propagation.
