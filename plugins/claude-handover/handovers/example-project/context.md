# Example Project — Handover Context

> This is a sample handover to demonstrate the format. All names, companies, and details are fictional. Delete this project whenever you want — run `/handover update example-project` or just delete the folder.

## Priorities

1. **Webhook format migration** (highest priority)
   - Acme is switching from v1 to v2 webhook format on April 1st
   - Our payment-gateway service needs to handle both formats during the transition
   - PR is open but needs testing with real v2 payloads from Acme staging
   - Zoey (Spark) reviewed the PR and had some feedback — check the comments

2. **Retry logic improvements** (second priority)
   - Currently we retry failed webhooks 3 times with no backoff
   - Need to implement exponential backoff to avoid hammering Acme's API
   - Should be a straightforward change in the consumer service

3. **Dashboard metrics** (third priority, nice to have)
   - Casey wants a dashboard showing webhook success/failure rates
   - No ticket yet — Casey will create one after the webhook migration

## Blockers

### Acme staging returns 500s intermittently
Their staging environment is flaky. About 10% of requests fail with 500s. We've flagged it with Riley but no ETA on a fix. For now, just retry and don't panic when you see errors in logs.

## Gotchas & Tribal Knowledge

- **Test accounts only on prod.** Acme Prod is live but we're still onboarding. Only use test account IDs.
- **Staging services scale down.** The payment-gateway pods on staging scale to 0 overnight. If things seem broken in the morning, check if the pods are up.
- **The payment-api-tester uses prod URLs.** Be careful — only target test accounts.
- **Webhook signatures changed in v2.** The HMAC key is the same but the signing algorithm changed from SHA256 to SHA512. Easy to miss.
- **Weekly Acme sync call.** Jordan should be added to this.

## Architecture Notes

### Data flow — inbound (webhooks)
```
Acme sends webhook → payment-gateway validates signature → transforms to internal format → publishes to internal queue
```

### Data flow — outbound (API calls)
```
Internal payment request → payment-gateway → transforms to Acme format → calls Acme API
```

## Additional Notes

- Acme's API docs are at https://docs.example.com (login credentials in 1Password under "Acme API Docs")
- There's a shared Google Sheet tracking test transaction IDs — ask Casey for the link
