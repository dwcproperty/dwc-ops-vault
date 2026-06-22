# 2026-06-22 — LeadSimple SMS Send Fix (`from`/`to` numbers)

Recurring failure fixed: sending an SMS via the LeadSimple MCP connector
(`send_text_message`) always returned `400 Bad Request — from is missing, to is
missing`, regardless of which contact/deal was passed. Now diagnosed, fixed, and
confirmed with one live send. Work on the box at `/root/dwc-mcp-servers/leadsimple/`.

## The symptom
- Tool: `send_text_message` on the LeadSimple MCP connector.
- Both attempts (contact only; contact + deal) failed identically:
  `LeadSimple API error: 400 Bad Request — from is missing, to is missing`.
- The tool's schema only exposed `contact_id`, `body`, optional `deal_id` — no
  number/channel field anywhere.

## Root cause
`POST /rest/text_messages` **requires explicit `from` and `to` phone numbers and
does NOT derive them from `contact_id`.** The connector was sending
`{contact_id, body, deal_id}` — so both `from` and `to` were simply never sent.

Two distinct gaps:
1. **No `from`.** The API needs the account's **SMS-enabled** number. That number
   is **`+14693005446`** (the DWC texting line — confirmed by 17 existing inbound
   text threads on it). `+14695359767` is **voice-only** (calls, no texts).
2. **No `to`.** It relied on `contact_id` to imply the recipient; the API ignores
   `contact_id` for routing. Compounding it: this REST key is **Unauthorized on
   `GET /rest/contacts/<id>`**, so the connector can't even look up a contact's
   phone — `to` must be passed in explicitly.

> Note: texting *had* worked inbound before — texts are filed under
> `kind=phone` in `/rest/conversations` (subject `"Text from (XXX) XXX-XXXX"`),
> which is why a naive "any text conversations?" check looked empty.

## The fix (applied + live)
Edited `/root/dwc-mcp-servers/leadsimple/index.js` → `send_text_message`:
- Added a constant `LS_SMS_FROM` (env `LS_SMS_FROM`, default `+14693005446`).
- Tool now POSTs `{ from, to, body, contact_id, deal_id }`.
- Schema now has a **required `to`** (E.164, e.g. `+12149247097`), optional
  `from` override; `contact_id`/`deal_id` kept for association only.
- Added `LS_SMS_FROM=+14693005446` to `.env`.
- `node --check` clean → `pm2 restart leadsimple-mcp` → `pm2 save`. Live.

**No LeadSimple account/config change was needed** — the SMS number already
existed; the connector just wasn't using it.

### Working request body
```json
{ "from": "+14693005446", "to": "+12149247097", "body": "...",
  "contact_id": "<uuid optional>", "deal_id": "<uuid optional>" }
```

### Going forward
Call `send_text_message` with **`to`** (E.164) + **`body`**. `from` defaults to the
DWC number; `contact_id`/`deal_id` optional. (claude.ai connector schema refreshes
to show the new `to` field after reconnect.)

## Confirmation — one live test send (to Madison Bowers)
`POST /rest/text_messages` → **`201 Created`**, verbatim:
```json
{"data":{"created_at":"2026-06-22T21:17:22Z","updated_at":"2026-06-22T21:17:22Z",
"id":"b43120c8-7714-4537-90a9-ce1337df42a9","to":"+12149247097","from":"+14693005446",
"description":null,"body":null,"direction":"outbound","manually_added":true}}
```
Created a real outbound thread, visible in `/rest/conversations`:
```
2026-06-22T21:17:22Z  kind=phone  inbox="+14693005446"  subj='Text to (214) 924-7097'
```
Body sent (re Madison's full Gmail inbox bouncing DWC emails):
> "Hi Madison, this is DWC Property Group. Quick heads-up: the emails we've been
> sending you keep getting kicked back because your inbox is full. When you get a
> chance, please clear some space so you don't miss anything important from us. Thanks!"

## Caveats
- `manually_added:true` = "created via API," **not** "undelivered." It produces a
  real outbound text thread identical to genuine ones.
- `body:null` in the create response = the API just doesn't echo body back.
- The scoped key **can't read delivery receipts** (`GET /rest/text_messages/<id>`
  → 403; no `GET` on the collection). For absolute proof-of-delivery, check the
  thread in the LeadSimple UI (Madison's conversation, 4:17 PM Central outbound)
  or watch for a reply.

## State after session
- `leadsimple-mcp` connector live with the SMS fix (pm2, restarted + saved).
- API quirks (required `from`/`to`, the `+14693005446` sender, scope limits)
  saved to agent memory `reference_leadsimple_api_quirks.md` so it won't be
  re-diagnosed.

Related: earlier this session — built the domain-wide **Gmail MCP** server (keyless
DWD, all 5 dwcproperty.com mailboxes) which is how the Madison full-inbox bounce
was found in the first place.

---

## Update (same day) — auto-resolve `to`; Cowork stale-schema cause

A later send attempt **through the Cowork app** failed again with `to is missing`,
even though the connector code was already fixed. Two findings:

**Why Cowork still failed:** Cowork/claude.ai was holding the **old cached tool
schema** (no destination field), so the model had nowhere to put a number.
Editing the connector + `pm2 restart leadsimple-mcp` does NOT refresh a connected
client's schema — it needs a **reconnect**. (Confirmed the user's hunch that they
never reset the app after the earlier repair.)

**Better fix — server-side number resolution** (so it works regardless of schema
cache or whether the model passes a number). `send_text_message` now
**auto-resolves `to`** via `resolveRecipientNumber()`:
1. explicit `to` (E.164) → use it;
2. else `deal_id` → `GET /deals/<id>` whose `contacts[]` carry `phone_numbers`
   (match the given `contact_id`, else first contact with a phone) — 1 lookup;
3. else `contact_id` → scan `GET /contacts?per_page=100` pages (up to 20). The key
   **ignores** `id`/`contact_id`/`ids[]` filters, so a bounded page scan is the only
   way (≈15 pages for 1,471 contacts). Note `GET /contacts` LIST works and carries
   `phone_numbers`, even though `GET /contacts/<id>` is 403.

Only `body` is required now; pass any one of `to` / `deal_id` / `contact_id`.

**Verified read-only** (no live send — Alise's note already went via Rentvine):
- Madison via deal+contact → `+12149247097` (via deal)
- Alise via contact only → `+13102796979` (via contact scan, page 8)
- explicit `to` → passthrough

**To use from Cowork:** retrying should now work (server resolves the number from
the contact/deal even with the stale schema). For the cleanest result, **reconnect
the LeadSimple connector once** so it also picks up the refreshed schema.

Files: `/root/dwc-mcp-servers/leadsimple/index.js` (`resolveRecipientNumber` +
`send_text_message` case + schema), restarted via `pm2 restart leadsimple-mcp`.
