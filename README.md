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

**v0.1.0 — early preview.**

Implemented:
- Stage 1 (initial reminder) end-to-end
- AI generator + static fallback
- Cron job + send log

Planned for v0.2.0+:
- Stage 2 (24 h follow-up) and Stage 3 (72 h with auto-generated coupon)
- Low-stock urgency email
- Observer to stop sends on order placement
- Recovery link controller + token-signed URL
- Unsubscribe controller
- Admin grid (send log listing + test-send button)

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
