# Account: balance, usage, billing, keys

All of these need the key (use a script) and none of them charge the balance.

- Balance: `POST /api/check-balance` → `usd_balance`, `nano_balance`. Run once after the plan is approved and compare with the high estimate.
- Usage: `GET /api/v1/usage?from=YYYY-MM-DD&to=YYYY-MM-DD&group_by=...` → spend, requests and tokens for the key (UTC days, range ≤ 366 days).
- Per-request billing: `GET /api/v1/usage/requests/{request_id}` - the charge for one request, available for about 24 hours; the request ID comes from the `X-Request-ID` response header. A 404 shortly after the call can mean "not recorded yet", not "free".
- Subscription quotas (if the user has a subscription): `GET /api/subscription/v1/usage`.
- Pricing overview for humans: `https://nano-gpt.com/pricing`.

## Scoped keys (recommendation to the user, not something scripts do)
- NanoGPT supports multiple named, revocable inference keys and a separate Management API (`https://docs.nano-gpt.com/api-reference/management-api.md`) for creating keys programmatically.
- Suggest the user put a dedicated, revocable key with a spend limit in `nanogpt.key` rather than their main key.
- Management tokens are a different credential type and are never needed by this skill.
