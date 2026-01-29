# How Purchases Work

## Data Model

```
purchases ←——many-to-many——→ wallpapers
    │
    ├── token (UUID)
    ├── stripeSessionId
    ├── email
    └── status
```

When a purchase is created, it gets linked to all wallpapers.

## Flow

```
1. User clicks "Buy Full Pack — $5"
         │
         ▼
2. POST /api/checkout
   └── Creates Stripe Checkout session
   └── Redirects to Stripe
         │
         ▼
3. User pays on Stripe
         │
         ▼
4. Stripe redirects to /success?session_id=cs_xxx
         │
         ├─────────────────────────────────┐
         ▼                                 ▼
5a. Webhook fires                    5b. Success page calls
    POST /api/webhook/stripe             POST /api/verify-purchase
         │                                 │
         └────────┬────────────────────────┘
                  ▼
6. createPurchase()
   └── Generate UUID token
   └── Create purchase record
   └── Link to all wallpapers
         │
         ▼
7. Token saved to localStorage
         │
         ▼
8. Future page loads:
   └── db.useQuery({ wallpapers: {...} }, { ruleParams: { token } })
         │
         ▼
9. InstantDB permission check:
   "ruleParams.token in data.ref('purchases.token')"
         │
         ├── Token valid → return fullResUrl
         └── Token invalid → omit fullResUrl
```

## Key Points

- **Token** — UUID stored in localStorage, proves ownership
- **ruleParams** — passes token to InstantDB permission rules
- **Field-level permission** — `fullResUrl` only returned if token matches a linked purchase
- **Dual creation path** — webhook OR verify-purchase API (whichever runs first)
- **Idempotent** — both paths check for existing purchase before creating
