# Sprint WS — WhatsApp & SMS Notification Channels

## Overview

| Field | Value |
|-------|-------|
| Sprint | Sprint WS — WhatsApp & SMS |
| Depends on | S6-02 (FCM push already wired), S8-00 (Agro Exchange match events exist) |
| Repos affected | `mombongo-functions`, `mombongo-web`, `mombongo-admin` |
| Total estimate | 10–14h |

## Context

FCM push notifications (S6-02) only reach users who have the app installed and granted notification permission. In the DRC market, a large share of users operate on low-end Android devices with poor data and limited app engagement. WhatsApp has near-100% penetration in DRC urban areas and SMS reaches everyone with a SIM card.

This sprint adds two complementary notification channels:

| Channel | Best for | Reach |
|---------|----------|-------|
| **WhatsApp** (Business API) | Rich messages, images, CTAs, templates | High-data users, merchants, cooperatives |
| **SMS** | Critical alerts, OTP, low-connectivity areas | All SIM card holders, farmers in rural areas |

## Provider decisions

### WhatsApp — Twilio
- Twilio's WhatsApp Business API is the most battle-tested integration.
- Provides template-based messaging (required for outbound WhatsApp Business messages).
- Environment vars: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WHATSAPP_FROM` (e.g. `whatsapp:+14155238886` for sandbox or your approved number).

### SMS — Africa's Talking
- Best SMS coverage in sub-Saharan Africa; direct carrier connections in DRC.
- Supports Lingala/French characters (UTF-8 SMS).
- Environment vars: `AT_API_KEY`, `AT_USERNAME`, `AT_SENDER_ID` (optional; defaults to shortcode).
- Alternative: Twilio SMS (same credentials, slightly higher cost in DRC).

---

## Stories

| Story | Title | Estimate |
|-------|-------|----------|
| [SWS-01](SWS-01-functions-notification-service.md) | Notification service CF + provider integrations | 4h |
| [SWS-02](SWS-02-user-preferences.md) | User notification preferences (web + CF) | 3h |
| [SWS-03](SWS-03-triggered-notifications.md) | Trigger notifications on platform events | 3h |
| [SWS-04](SWS-04-admin-broadcast.md) | Admin broadcast panel (send to segment) | 2–4h |
| [SWS-05](SWS-05-morning-price-whatsapp.md) | Morning price push via WhatsApp/SMS | 2h |

---

## Manual setup required (before deployment)

See [MANUAL_SETUP.md](../../../MANUAL_SETUP.md) section "WhatsApp & SMS" (to be added after this sprint is approved).

Key steps:
1. Create Twilio account → enable WhatsApp Sandbox (or apply for a Business number)
2. Register WhatsApp message templates (24h approval by Meta)
3. Create Africa's Talking account → get API key + DRC short code
4. Set Firebase Functions secrets (see SWS-01)

---

## Notification event matrix

| Event | WhatsApp | SMS | FCM | Story |
|-------|----------|-----|-----|-------|
| Transaction confirmed (deposit/withdrawal) | ✅ | ✅ | ✅ | SWS-03 |
| Agro Exchange match found | ✅ | — | ✅ | SWS-03 |
| Contract signed by other party | ✅ | ✅ | ✅ | SWS-03 |
| Escrow funded (seller) | ✅ | ✅ | ✅ | SWS-03 |
| Delivery confirmed → funds released | ✅ | ✅ | ✅ | SWS-03 |
| Financing application status change | ✅ | ✅ | ✅ | SWS-03 |
| KYC approved/rejected | — | ✅ | ✅ | SWS-03 |
| OTP / verification code | — | ✅ | — | SWS-03 |
| Admin broadcast (marketing) | ✅ | ✅ | ✅ | SWS-04 |
| Harvest due reminder (farmer) | ✅ | ✅ | ✅ | SWS-03 |
| **Morning price push (daily 06h30)** | ✅ | fallback | ✅ | **SWS-05** |
