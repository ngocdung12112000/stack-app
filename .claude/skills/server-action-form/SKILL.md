---
name: server-action-form
description: Build a form or mutation in the StackShop Next.js apps using react-hook-form + zod on the client, a Server Action (or route handler) that re-validates with the same zod schema and calls @stackshop/api-client, ProblemDetails-to-field error mapping, revalidateTag/revalidatePath, and optimistic UI. Use for login/register, address forms, add-to-cart, checkout steps, reviews, admin create/edit forms and any button that mutates data.
---

# Form + Server Action

## 1. Schema first (shared)

`apps/<app>/lib/schemas/<thing>.ts`:

```ts
export const addressSchema = z.object({
  recipientName: z.string().min(1).max(100),
  phone: z.string().regex(/^\+[1-9]\d{6,14}$/, 'invalidPhone'),
  // …mirror the BR-n limits from the spec exactly
});
export type AddressInput = z.infer<typeof addressSchema>;
```

The client form and the Server Action use the **same schema**. Message keys are next-intl keys, not English strings. For type-dependent forms (e.g. promotion types), use `z.discriminatedUnion`.

## 2. Server Action

`app/(group)/<route>/actions.ts`:

```ts
'use server';
export async function saveAddress(input: AddressInput): Promise<ActionResult<{ id: string }>> {
  const parsed = addressSchema.safeParse(input);
  if (!parsed.success) return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors };
  const res = await api().addresses.create(parsed.data);          // server-only client, attaches cookie token
  if (!res.ok) return fromProblem(res.problem);                   // shared helper: code → message key, errors → fieldErrors
  revalidateTag('addresses');                                     // or revalidatePath
  return { ok: true, data: { id: res.data.id } };
}
```

Rules:
- Re-validate on the server, and never trust client data.
- `ActionResult` is the shared discriminated union (`ok | fieldErrors | formError(code)`).
- **ProblemDetails mapping:** `errors` → field errors, `code` → a translated form-level message (`errors.ORDERING_PRICE_CHANGED`). Never surface raw `detail`.
- Mutations that need special UX handle their codes explicitly, e.g. `PRICE_CHANGED` → confirm dialog, `CART_INSUFFICIENT_STOCK` → show `available`.
- Checkout and payment calls send the `Idempotency-Key` that was generated once per checkout session and reused on retries.
- Auth actions (login/refresh/logout) set or clear the `at`/`rt` httpOnly cookies here, and never return tokens to the client.
- Admin edits send `If-Match` with the ETag loaded with the form. On 412, show the "changed by someone else, reload?" dialog.

## 3. Client form

- `useForm({ resolver: zodResolver(schema), defaultValues })` + shadcn `<Form>`, `FormField`, `FormMessage`.
- Submit with `useTransition`. Disable the submit button while pending (this also prevents double submit).
- Apply `fieldErrors` from the result with `form.setError`. Show the form-level error in an `Alert`. On success, show a Sonner toast and redirect/close the dialog.
- Optimistic UI for quick toggles (add to cart, wishlist heart, mark read): `useOptimistic`, then reconcile with the result and roll back + toast on failure.
- Accessibility: every input has a label, errors are linked via `aria-describedby`, and focus moves to the first invalid field.

## 4. Tests

- Vitest: schema edge cases (each BR limit), and the form showing the server field errors from a mocked action result.
- Playwright: the spec's journey, including one error path (e.g. 409 → message shown).

## 5. Handoff

**backend-engineer:** if validation limits differ between the spec and the API, or a needed error code is missing.
