# Inspections Portal — Operator SOP

How to publish a DWC-branded inspection report from an OnSightPros URL and email the owner the link. End-to-end in ~60–90 seconds.

**Also in Notion:** [Publish an Inspection Report (Inspections Portal)](https://www.notion.so/36523e9aa5a381628be4d3be26936969) — keep both copies in sync if you edit one.

---

## When to use this

Whenever an OnSightPros inspection (move-in, move-out, or periodic) is ready to share with the owner. The portal at **inspections.collincountyrent.com** mirrors the report under DWC's brand — same photos, same 360° walkthroughs, plus a PDF cover letter — so the owner never sees the OnSightPros chrome.

## Before you start

You need to be on the Google sign-in allowlist for the portal. Currently allowed:

- `office@dwcproperty.com`
- `mary@dwcproperty.com`

To add a teammate, ping the developer (one-line edit in `.env`, no GCP change needed).

---

## The workflow

### 1. Get the OnSight URL

Sign in to [app.onsightpros.com](https://app.onsightpros.com) → open the report you want to publish → copy the URL from your browser's address bar. It looks like:

```
https://app.onsightpros.com/report/1D1E3C6F-FA98-6843-9B25-99B7A55B8A72
```

### 2. Import it into the DWC portal

Go to **[inspections.collincountyrent.com/admin](https://inspections.collincountyrent.com/admin)** and sign in with your DWC Google account.

Paste the OnSight URL into the **Import an inspection** box and click **Import**.

A progress bar shows live status — signing in to OnSight, scraping every photo and 360°, downloading media, publishing. **Usually 60–90 seconds.** Larger reports (200+ photos) may take up to 2 minutes.

### 3. Send it to the owner

When the import finishes, the new inspection appears in the **Published inspections** table on the same page. Click the **Email owner** button on its row, confirm the recipient email, and click send.

The owner receives a DWC-branded email from `office@dwcproperty.com` with a link to the report. The email also points them to the **Download PDF** button on the report page, which gives them a printable version with a custom cover letter.

Done. You don't need to copy/paste the URL anywhere.

---

## What each admin button does

| Button | What it does |
|---|---|
| **Re-scrape** | Re-runs the OnSight import. All rooms and photos refresh. Photos already on disk are reused (hash dedupe), so re-scrape is fast. Use this if OnSight added or edited photos after the first import. |
| **Email owner** | Sends the branded email with the report link. Subject and copy are fixed; you just confirm the recipient address. |
| **Archive** | Hides the inspection from the admin list. Nothing is deleted — the public URL keeps working and the inspection can be restored from the database. |

---

## If something goes wrong

### The import fails

Click **Re-scrape**. Most failures are transient (network blip on OnSight's side, slow S3 response).

If it fails twice in a row, the OnSight session has likely expired or hit a CAPTCHA. **Ping the developer** to refresh the stored credentials. The **Recent import jobs** table at the bottom of the admin page shows the exact failure step.

### The wrong address shows up

The address comes from OnSightPros's property record. If it's wrong on the DWC page, fix it in OnSightPros first, then click **Re-scrape** here.

### Owner can't open the link

The public report URL works without any sign-in. If the owner reports a problem, ask which page loaded — if they got a "Not authorized" page, they probably clicked the `/admin` link by mistake. The owner-facing link looks like:
`https://inspections.collincountyrent.com/<address-slug>`

---

## What the owner sees

- A DWC-branded page with the property address up top
- An **Overall condition** tile (Excellent / Good / Fair / Needs Attention — derived from the inspector's attention flags)
- A dashboard strip: Rooms · Photos · 360° Walkthroughs · Items Needing Attention
- A left-rail TOC with red badges showing how many flagged items each room has
- An **Items Needing Attention** section at the top with cards for every flagged photo
- Room-by-room walkthrough with photos, condition notes, and interactive 360° viewers
- **Download PDF** button → multi-page PDF with a custom cover letter on page 1, then the full report

---

## Tools & links

- **Admin:** [inspections.collincountyrent.com/admin](https://inspections.collincountyrent.com/admin)
- **OnSightPros:** [app.onsightpros.com](https://app.onsightpros.com)
- **Example published report:** [inspections.collincountyrent.com/1707-aleia-cove](https://inspections.collincountyrent.com/1707-aleia-cove)
- **Notion copy of this SOP:** https://www.notion.so/36523e9aa5a381628be4d3be26936969

## Notes

The report page intentionally has `noindex` set, so Google won't surface it in search results. Each inspection has a unique URL based on the address (e.g. `/1707-aleia-cove`). If a property gets inspected again later, the new inspection lands at a dated slug (e.g. `/1707-aleia-cove-2026-09-15`) so the prior report stays reachable.

---

## See also

- [[Handoff — paused 2026-05-19]] — the original design + paused-state handoff for this build
- `/root/dwc-mcp-servers/inspections-portal/README.md` on the server — operator-level technical reference (credentials, log paths, restart commands)
