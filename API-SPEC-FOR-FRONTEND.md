# GP Ticketing API — Integration Spec for Frontend

For the developer(s) building the customer-facing side on the
genesispeptides.com (Next.js) frontend. The backend is already built and
installed on wp.genesispeptides.com. You do not need WordPress access to
integrate this, just these four endpoints.

Base URL: `https://wp.genesispeptides.com/wp-json/gp/v1`

Before going live, confirm with whoever manages the WordPress install that
your frontend's actual domain(s) are in the CORS allow-list
(`includes/class-gpst-rest.php`, `allowed_origins()`). Requests from any
other origin will be blocked by the browser.

---

## 1. Get ticket categories (for a dropdown)

`GET /categories`

No auth, no params.

Response `200`:
```json
[
  { "value": "order_status", "label": "Order Status / Tracking" },
  { "value": "payment", "label": "Payment Issue" },
  { "value": "stock_question", "label": "Product / Stock Question" },
  { "value": "coa_lab", "label": "Certificate of Analysis / Lab Test" },
  { "value": "return_refund", "label": "Return or Refund" },
  { "value": "account", "label": "Account Access" },
  { "value": "other", "label": "Other" }
]
```

---

## 2. Submit a ticket

`POST /tickets`

Body (JSON):
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "category": "order_status",
  "order_number": "10432",
  "message": "My order hasn't arrived yet."
}
```

`name`, `email`, `category`, `message` are required. `category` must be one
of the `value` fields from `/categories`. `order_number` is optional.

Response `201`:
```json
{
  "ticket_id": 482,
  "token": "9f2c1a...(32 hex chars)",
  "priority": "urgent",
  "status": "open"
}
```

**Store the `ticket_id` and `token` client-side** (e.g. in the account
session if the customer is logged in, or show them a "save this link"
message if not) — they're required to check status or reply later, and
there's no way to recover them otherwise. The customer also receives them
baked into a link in their confirmation email automatically, so this is a
convenience for in-app status checking, not the only way to get back in.

`priority` is informational only, useful if you want to show the customer
something like "we'll prioritize this" copy for urgent categories, but it
has no effect on what they can do.

Error responses (`400`): `{ "error": "gpst_invalid", "message": "..." }`

---

## 3. Get ticket status and thread

`GET /tickets/{id}?token={token}`

Response `200`:
```json
{
  "ticket_id": 482,
  "title": "Jane Doe - 2026-08-25 14:03",
  "status": "waiting_on_customer",
  "status_label": "Waiting on Customer",
  "category": "order_status",
  "category_label": "Order Status / Tracking",
  "priority": "urgent",
  "order_number": "10432",
  "is_closed": false,
  "thread": [
    { "author": "customer", "message": "My order hasn't arrived yet.", "date": "2026-08-25 14:03:00" },
    { "author": "staff", "message": "Checking on this now, one moment.", "date": "2026-08-25 15:10:00" }
  ]
}
```

`author` is either `"customer"` or `"staff"`, use it to align message
bubbles left/right the way you'd build any chat thread.

`403` if the token doesn't match the ticket, `404` if the ticket doesn't
exist. Treat both as "show the same generic access-denied state," don't
distinguish them in the UI, no reason to reveal which ticket IDs exist.

---

## 4. Customer replies to a ticket

`POST /tickets/{id}/reply`

Body (JSON):
```json
{
  "token": "9f2c1a...",
  "message": "Still nothing, it's been a week."
}
```

Response `200`: `{ "success": true }`

`409` if the ticket is already resolved/closed (message: ticket is closed,
tell them to submit a new one instead). `403` for a bad token. `400` if
message is empty.

---

## Suggested UI flow

1. Support/contact page: form with Name, Email, Category (from `/categories`),
   Order Number (optional), Message. Submit hits endpoint 2.
2. On success, show a confirmation and (if you want in-app tracking rather
   than relying purely on email) redirect to a status view seeded with the
   returned `ticket_id`/`token`.
3. Status view: call endpoint 3, render the thread, show a reply box unless
   `is_closed` is true, wired to endpoint 4.

None of this requires the customer to have an account, though if your
account system already has their email on file, you could list their past
tickets by storing `ticket_id`/`token` pairs against their account record on
your side, the API itself has no concept of accounts, it's token-based
per ticket by design so it doesn't need to know anything about your auth
system.
