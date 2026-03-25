# Webhook v2 Testing Update

**Date:** 2026-03-27
**Author:** Jordan

Tested the v2 webhook format against Acme staging. Most event types work correctly. Found one issue: the `refund` event type has a different field name in v2 (`refund_amount` instead of `amount`). Created a PR to handle both.

Zoey reviewed and approved. Merging tomorrow.
