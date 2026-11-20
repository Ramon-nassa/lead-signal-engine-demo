# Design: ad-platform signals (Meta + Google), CRM-agnostic

Written before the code (18/11/2026). Method: events first, then the states one lead
goes through per destination, then the rules that must always hold (invariants).

## 1. Decisions

1. **One system, destinations as a parameter.** `AD_DESTINATIONS=meta,google`. A lead is
   not "a Meta lead" or "a Google lead": the engine sends the qualified signal to every
   enabled destination that can match it.
2. **The engine sends, not the CRM.** Click ids are stored on the lead at intake, and the
   engine calls Meta CAPI and Google directly. Any CRM (or none) works; no native
   HubSpot to Google Ads sync.
3. **Google goes through the Data Manager API** (`datamanager.googleapis.com/v1/events:ingest`).
   Since 15/06/2026 the Google Ads API refuses new adopters of `UploadClickConversions`.
4. **Capture works on any page.** A small script (`/lead-capture.js`) reads click ids from
   the URL and first-party cookies, keeps them for 90 days, and adds them to every form on
   submit. It does not depend on the page builder (Shopify, Webflow, WordPress...).

## 2. Events

| Event | Comes from | Carries |
|---|---|---|
| `AdClicked` | the visitor lands with `gclid` / `gbraid` / `wbraid` / `fbclid` in the URL | click ids, UTMs |
| `FormSubmitted` | any form on the page (script) or a direct POST | contact fields + stored click ids + `event_id` |
| `MetaLeadSubmitted` | Meta Lead Ads webhook | `leadgen_id` |
| `GoogleLeadSubmitted` | Google lead form webhook | `lead_id`, `gcl_id` |
| `LeadQualified` | score >= 70, or a person qualifies it | `event_id` |
| `SignalAccepted` / `SignalFailed` | Meta or Google answer | request id, error |

## 3. States of one signal (per lead, per destination)

```
not_eligible ── (no consent, or not qualified, or nothing to match on)
pending ──send──> sent
   │  └─transient error──> retrying ──> sent
   └─permanent error / retries used up──> dead_letter ──retry-failed──> sent
```

## 4. Invariants (must always hold)

1. A destination never counts the same lead twice: Meta dedupes on `event_id`,
   Google on `transactionId = event_id`, and the local ledger sends each step once.
2. A signal is sent only for a qualified lead with ad consent.
3. Google gets a signal only when it can match it: a Google click id, a Google lead form
   `lead_id`, or (if `GADS_SEND_WITHOUT_CLICK_ID=true`) hashed email or phone.
4. Contact data leaves the engine only SHA-256 hashed; click ids are sent as is.
5. One destination failing does not block the other, and does not undo the CRM or reply.
6. Account ids and tokens never reach a trace.
