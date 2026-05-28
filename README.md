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

**v0.10.0 — admin test-send button.**

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
- **One-click unsubscribe**: every email carries an unsubscribe link that flips the `unsubscribed` flag on every log row for the (customer_email, store_id) pair. Both finders consult that flag before yielding candidates.
- **Admin send-log grid** at *Marketing → Communications → Abandoned Cart Log* — filterable/sortable columns (Type, Recipient, Store, Status, AI?, Coupon, Recovered At, Unsubscribed?), CSV/Excel export, mass-delete and mass-mark-unrecovered actions.
- **Personalized coupon codes**: codes are prefixed with the customer's sanitized first name (or email local-part when no name is on file), e.g. `VERONICA-AB3F` instead of `0GK2IY2HCNVB`. Magento's coupon generator still guarantees uniqueness via the random suffix.
- **Product images + redesigned emails**: every email shows a cart-items grid (thumbnail · name · qty · line price) above the AI body copy. Card layout on a soft canvas, dashed coupon banner, primary CTA. Low-stock template gets a red urgency pill and red CTA.
- **Style rotation**: each send randomly picks one of 7 rhetorical approaches (curious question, vivid observation, playful, brief, scene-setting, friend-texting, direct statement) so the same customer reading stages 1/2/3 doesn't see the same "Hey {name}" pattern three times.
- **Admin test-send button** on the send-log grid. Dispatches one preview email of the selected type to any recipient using a synthetic sample cart — no coupon minted, no log row written.

Planned for v1.0.0+:
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
