# Magebit Abandoned Cart

AI brand-voice abandoned-cart recovery emails for Magento 2 + Hyvä, powered by Google Gemini.

## Features

- Cron-driven scanner that finds abandoned carts and dispatches recovery emails
- Per-stage AI-generated subject + preheader + body in your configured brand voice (via Google Gemini 2.5 Flash, free tier)
- Static-template fallback when the API is unavailable or unconfigured — pipeline never silently drops a customer
- Dedup-safe send log with `UNIQUE(quote_id, stage_key)`
- Per-store-view brand voice (name, voice description, tone, locale)
- Admin configuration tree under **Stores → Configuration → Magebit → Abandoned Cart**
- PHPStan level 9 + PHPCS (Magento2 / PSR-12 / Slevomat) clean

## Status

**v0.4.0 — four email types, four channels.**

Implemented:
- Three-stage abandoned-cart cadence: stage 1 reminder, stage 2 follow-up, stage 3 with auto-generated coupon
- **Low-stock urgency email** — separate cron, fires daily when any cart item drops to qty ≤ threshold but is still > 0; carries its own coupon from a separate admin-selected rule
- AI generator (Google Gemini) + static template fallback per email type
- Cron-driven scanner with send-log dedup keyed on `(quote_id, stage_key)` — stage keys for low-stock embed the date so the unique constraint enforces a natural daily cap
- `CouponIssuer` mints unique codes from any admin-selected cart price rule (requires `use_auto_generation=1` on the rule)
- Generated coupon is rendered as a prominent banner AND mentioned naturally in AI body copy
- `sales_order_place_after` observer that suppresses future sends once the customer buys
- Token-signed recovery link in every email — restores the quote into the visitor's session and redirects to `/checkout/cart`
- API key redaction in error logs

Planned for v0.5.0+:
- Unsubscribe controller
- Admin grid (send log listing + test-send button)
- Unit + integration tests

## Requirements

- Magento Open Source / Commerce 2.4.8 or newer
- PHP 8.3+
- Google Gemini API key (free at [aistudio.google.com](https://aistudio.google.com)) — optional, falls back to static templates without one

## Installation

```bash
composer config repositories.magebit-abandoned-cart vcs https://github.com/flowcheckdnb-cpu/magento2-abandoned-cart
composer require magebit/module-abandoned-cart:^0.1
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:clean
```

## Configuration

**Stores → Configuration → Magebit → Abandoned Cart:**

| Group | Fields |
|---|---|
| General | Enable module, sender identity, unsubscribe route |
| Stages | Per-stage delay + email template |
| Low-stock | Threshold, template, coupon rule, frequency cap (planned) |
| Brand voice | Brand name, voice description, tone, locale |
| Gemini API | Encrypted API key, model selector, kill-switch, request timeout |

Then ensure two `trans_email/ident_*/name` config rows are populated (Magento requires both `email` and `name` for sender identities) and run cron — either via the natural heartbeat or:

```bash
bin/magento cron:run --group=default
```

## Verifying

Backdate an active quote into the stage-1 window and force-run cron. The email should land at your configured SMTP destination (e.g. Mailpit in local development).

```sql
UPDATE quote SET is_active=1, updated_at=NOW() - INTERVAL 2 HOUR WHERE entity_id=...;
```

## License

OSL-3.0 — see [LICENSE](LICENSE).
