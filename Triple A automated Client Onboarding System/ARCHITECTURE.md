# Architecture and Technical Notes
![Full system, all seven workflows](screenshots/All%20workflows.png)

This document is the detailed companion to the README. It exists so that six months from now, without any memory of building this, you can open this file and reconstruct exactly how each piece works, not just what it does.

Where a formula, expression, or field name mattered to getting something working, it is written out here exactly as it was used. Where a bug took real debugging to solve, the mechanical reason is explained, not just the symptom.

---

## Data model

### Deals table
Pre existing sales pipeline table. Not built as part of this project. The only thing this system depends on from it is a Deal Stage field that can equal "Closed Won," and a Deal Value field that Onboarding pulls in as a linked lookup.

### Onboarding table
One record per client. Created only when a Deal reaches Closed Won. This is the table every workflow reads from and writes to.

Key fields:

- **Client Name** (text)
- **Contact Name** (text)
- **Contact Email** (text)
- **Status** (single select) — the full list of values: Deal Won, Contract Sent, Contract Signed, Payment Pending, Payment Confirmed, Intake Received, Provisioning, Provisioned, Kickoff Pending, Kickoff Scheduled, Onboarding Complete, and any `Blocked – [reason]` value as needed.
- **Last Status Change** (date/time, updated automatically whenever Status changes) — this is what the Sweeper's hours-elapsed calculation depends on entirely.
- **Deal Value (from Linked Deal)** — lookup field, used to set the exact payment amount dynamically rather than a hardcoded price.
- **PandaDoc Document ID** (text) — saved after Workflow 2 creates the contract, used by Workflow 3 to match the signature webhook back to the correct record.
- **Payment Link** (URL)
- **Paddle Transaction ID** (text) — saved after Workflow 3 creates the transaction, used by Workflow 4 to match the payment webhook back to the correct record.

### Activity Log table
Linked to Onboarding. Every workflow writes to this table at nearly every step. Fields:

- **Client** (linked record, points to Onboarding)
- **Event** (text) — a short label such as "Contract Signed" or "Nudge Sent – Payment Pending"
- **Note** (text) — free text, used for extra detail. Critically, this is where the Sweeper's dedup logic looks for the client's name, since the linked Client field cannot be easily text searched inside a Code node the way a plain string can.
- **Timestamp** (date/time)

The Sweeper's nudge log entries are written in this format for the Note field:
```
Client: [Client Name] | Stage: [Status Name] | hoursElapsed: [number]
```

### Intake Data table
Separate table, not fields on Onboarding directly. Populated through an Airtable form.

- **Client Name** (text, hidden on the public form, prefilled via URL)
- **Onboarding Record ID** (text, hidden on the public form, prefilled via URL) — this is the field that actually matters. It holds the Onboarding record's own Airtable record ID (the `rec...` string), and Workflow 5 uses this, not the client name, to find the matching Onboarding record. Record IDs are effectively impossible to guess or usefully tamper with, so even if the hidden field were exposed and edited, the worst outcome is a failed match that routes to a manual review alert, not a silent misattribution to the wrong client.
- Business Legal Name, Business Address, Primary Contact Phone, Branding Assets, Brand Colors/Style Notes, Tools They Currently Use, Team Members to Add, Preferred Kickoff Date, Anything Else We Should Know — all client facing, all visible on the form.

**On the hidden field mechanism specifically:** Airtable's form builder silently drops the value of any field manually toggled off in the field list, even if a prefill URL parameter targets it. The fields must stay toggled ON in the builder. To actually hide them from the person filling out the form while still submitting their prefilled value, the URL needs a paired `hide_FieldName=true` parameter alongside each `prefill_FieldName=value` parameter. Example structure:

```
https://airtable.com/[base]/[form]?prefill_Client%20Name=Coral%20Bay%20Realty&hide_Client%20Name=true&prefill_Onboarding%20Record%20ID=recXXXXXXXX&hide_Onboarding%20Record%20ID=true
```

Note the use of `%20` rather than `+` for spaces inside the field name portion of the parameter keys. Using `+` worked in isolated testing but broke once the link was sent through a real Gmail HTML email, because Gmail's link processing re-encodes `+` characters inconsistently, which corrupted the field name Airtable was trying to match against. `%20` survives that re-encoding safely.

---

## Workflow 1: Deal Won → Onboarding Record Creation
![Workflow 1 canvas](screenshots/1%20-%20Deal%20Won%20to%20Onboarding.png)

- **Trigger:** Airtable Trigger on Deals, watching a Last Modified Time formula field (the native "Trigger On" field options did not behave as expected, so a formula field was used as a reliable substitute).
- **Filter node** (not IF): passes only records where Deal Stage equals Closed Won.
- **Duplicate check:** Airtable Search node on Onboarding, formula:
```
AND({Client Name} = "{{ $json['Client Name'] }}", {Status} = "Deal Won")
```
followed by an empty-check gate before the Create step, so the same Deal Won event firing twice does not create two records.
- **Create Onboarding record**, then **create Activity Log entry**. This two step pattern, create/update the record followed immediately by a linked Activity Log entry, is used consistently across every workflow in this system.
- **Linked field pattern:** Airtable linked record fields require an array of record IDs, not plain text. The working expression pattern:
```
["{{ $('Filter').item.json.id }}"]
```
A JSON array literal wrapping a single expression, typed directly into the field. This exact pattern is reused for every linked field write in every workflow.

---

## Workflow 2: Contract Generation & Send
![Workflow 2 canvas](screenshots/2%20-%20Contract%20Generation%20and%20Send%20copy.png)

- **Trigger:** Airtable Trigger on Onboarding, Status equals Deal Won.
- **Auth note:** PandaDoc uses simple static Header Auth (`Authorization: API-Key {key}`, the literal word "API-Key," not "Bearer") since its API is built around single account API keys rather than OAuth2's expiring token and consent flow.
- **Async handling:** PandaDoc's create-document call returns immediately with a status of `document.uploaded`, not `document.draft`, along with an instruction to poll the Document Status endpoint. A polling loop was built: a Set node tracking attempt count, a Wait node, an HTTP GET status check, an incrementing Set node, and an IF node that proceeds once status equals `document.draft`, or routes to a failure path after five attempts.
- **Template bug, resolved:** tokens (Client.Name, Client.Email, Client.DealValue, Effective.Date) failed to populate when the source template was a PDF upload, even though the placeholder text looked correctly formatted. Switching the template source to a native Google Docs document, with the same dot notation tokens typed directly into the text, resolved this completely. The underlying cause is that PDF conversion can flatten or alter placeholder text in ways that break PandaDoc's token parser, even when nothing looks wrong visually.
- **Send step:** POST to `/documents/{id}/send`, body includes `"silent": false` to actually trigger the email to the recipient.
- **Error handling:** On Error set to Continue (using error output) on both the create and send HTTP nodes, routing failures to a status update of `Blocked – Contract Send Failed` plus an Activity Log entry.

---

## Workflow 3: Signature Received → Payment Trigger
![Workflow 3 canvas](screenshots/3%20-%20PandaDoc%20%E2%86%92%20Airtable%20%E2%86%92%20Paddle%20Onboarding.png)

- **Trigger:** Webhook node, method must be POST (PandaDoc always sends POST; the node defaulted to GET initially, which caused every incoming call to return a 404 since no route existed for that method).
- **Authentication:** set to None. PandaDoc does not send a checkable header for authentication in the way Header Auth expects. Instead it signs the payload body using the shared key via HMAC and includes the resulting signature in a response header, meant to be verified inside the workflow logic itself rather than checked at the node's auth gate. Setting Header Auth here caused every real request to be rejected with a 403, since the incoming header contains a computed signature hash, never the literal shared key.
- **Production vs Test URL:** the Test URL only listens for a single call while actively armed in the editor. For a webhook to work automatically with the workflow closed, PandaDoc must be pointed at the Production URL, and the workflow itself must be toggled Active.
- **Match logic:** Airtable Search on Onboarding:
```
{PandaDoc Document ID} = "{{ $json.data.id }}"
```
- **Paddle transaction, dynamic pricing:** rather than a pre-created catalog price, the transaction is built using a non-catalog price object defined directly on the request, so the exact deal value from Airtable can be passed in every time:
```json
{
  "items": [
    {
      "quantity": 1,
      "price": {
        "description": "Onboarding service fee - {{ $('Search Onboarding').item.json.fields['Client Name'] }}",
        "name": "Onboarding Service Fee",
        "unit_price": {
          "amount": "{{ $('Search Onboarding').item.json.fields['Deal Value (from Linked Deal)'][0] * 100 }}",
          "currency_code": "USD"
        },
        "product": {
          "name": "Agency Onboarding Service",
          "tax_category": "standard",
          "description": "Client onboarding and setup service"
        }
      }
    }
  ],
  "currency_code": "USD"
}
```
Amount is in the smallest currency unit, cents, hence multiplying the dollar value by 100.
- **Paddle account setup requirement:** a transaction cannot be created at all until a default checkout domain is set in the Paddle dashboard under Checkout Settings, and that domain must be an approved one. The sandbox account comes with `paddle.com` pre-approved, which is usable directly for testing.
- **Known limitation:** the resulting checkout URL only fully renders into a working payment page on a domain that has Paddle.js embedded on it. Since this project has no live front end, the checkout URL was confirmed to generate correctly via the API response, but does not render a functioning payment form when opened directly. This was treated as an accepted, documented limitation rather than something to build a workaround for.
- **Error branch:** On Error set to Continue (using error output) on the Paddle HTTP node. On failure, Status updates to `Blocked – Payment Link Failed`.
- **Connection level vs application level errors:** a DNS resolution failure (wrong hostname, request never reaches a server) was initially observed landing in the success output instead of the error branch, even with Continue (using error output) set correctly. A proper HTTP error response (wrong auth key, 401 from a real server) routed correctly to the error branch as expected. This is a known inconsistency in how some HTTP node versions classify a connection failure versus an application response failure, worth remembering if a future debugging session sees the same pattern.

---

## Workflow 4: Payment Confirmed → Intake Trigger
![Workflow 4 canvas](screenshots/4%20-%20Paddle%20Payment%20%E2%86%92%20Onboarding%20Intake.png)

- **Trigger:** Webhook node, POST, Authentication None, same reasoning as Workflow 3's webhook.
- **Paddle webhook registration:** done through Paddle's dashboard under Developer Tools, in two parts. First, a notification destination is created under Notifications, pointing at the n8n Production URL and selecting the relevant transaction event type. Second, real test events can be fired at that destination through the separate Simulations section, without needing a working checkout page.
- **Match logic:** Airtable Search on Onboarding:
```
{Paddle Transaction ID} = "{{ $json.data.id }}"
```
- **Email link, HTML formatted:** the Gmail node's message type must be set to HTML, otherwise an `<a href>` tag renders as literal visible code rather than a clickable link.

---

## Workflow 5: Intake Received → Provisioning
![Workflow 5 canvas](screenshots/5%20-%20Client%20Onboarding%20Provisioning.png)

- **Trigger:** Airtable Trigger on Intake Data, record created.
- **Match logic:** Airtable Search on Onboarding, matching against the hidden Onboarding Record ID field carried through from the intake form:
```
RECORD_ID() = "{{ $json.fields['Onboarding Record ID'] }}"
```
- **Slack channel naming:** Slack channel names cannot contain spaces or capital letters. Expression used to generate a valid name from the client name:
```
{{ 'client-' + $('Search Onboarding').item.json.fields['Client Name'].toLowerCase().replace(/\s+/g, '-') }}
```
- **Slack invite behavior:** creating a channel via the API does not make it appear in the creator's sidebar automatically, which initially looked like a bug. Slack's `conversations.invite` API method actually rejects an attempt to add the channel's own creator, returning `cant_invite_self`, because that identity is technically already a member. The real issue was UI visibility, not membership. The invite step in this workflow is used to add a different, actual team member to the new channel, which is both the correct real world use case and avoids the self-invite error entirely.
- **Slack scopes required:** `users:read` to populate a user picker, and `channels:manage` (or `groups:write` for private channels) to actually perform the invite. Missing scopes surface as a permissions error on the invite call specifically, not on channel creation.
- **Per step error branching:** Drive, Asana, and Slack each have independent On Error set to Continue (using error output), so a failure in one is reported with its own specific Blocked status and Activity Log entry, rather than one generic failure covering all three. Each failure message notes which of the earlier steps in the sequence succeeded before the failure occurred, so a human resolving it knows exactly how much manual cleanup is actually needed.

---

## Workflow 6: Welcome Email + Kickoff Scheduling
![Workflow 6 canvas](screenshots/6%20-%20Welcome%20Email%20+%20Kickoff%20Scheduling.png)

Two independent trigger branches inside one workflow file. n8n fully supports multiple trigger nodes in a single workflow; each operates completely independently and starts its own execution whenever it fires, with no dependency on the other branch.

**Branch A:** Airtable Trigger, Status equals Provisioned, sends welcome email with a Cal.com booking link, updates Status to Kickoff Pending.

**Branch B:** Webhook node listening for a Cal.com booking confirmation event, POST, Authentication None. Matches the booking back to the correct Onboarding record, updates Status to Kickoff Scheduled, posts a Slack notification.

The relationship between the two branches is conceptual, not mechanical. Branch B does not run because Branch A ran. It fires whenever Cal.com sends a booking event, regardless of what originally sent the client the link.

---

## Workflow 7: The Sweeper
![Workflow 7 canvas](screenshots/7%20-%20Onboarding%20SLA%20Nudge%20%26%20Escalate.png)

- **Trigger:** Schedule Trigger, every 2 hours.
- **Monitored statuses:** Contract Sent, Payment Pending, Payment Confirmed, Kickoff Pending. Chosen specifically because these are the only stages where the system is waiting on the client to take an action; internal automated steps and terminal or already-blocked states are deliberately excluded.
- **Fetch filter (Search Onboarding node):**
```
OR({Status}="Contract Sent", {Status}="Payment Pending", {Status}="Payment Confirmed", {Status}="Kickoff Pending")
```
- **Fetch filter (Search Activity Log node):**
```
FIND("Nudge Sent", {Event})
```
This pulls every nudge-related log entry across all clients in one bulk fetch. It is not meant to be scoped per client at this stage; the actual per-client, per-stage matching happens next, inside the Code node, since matching logic is far easier to write correctly in JavaScript than as a single static Airtable formula covering every client and stage combination at once.

- **Compute node (Code), full logic:**
```javascript
const now = new Date();
const records = $('Search Onboarding').all().map(i => i.json);
const logs = $('Search Activity Log').all().map(i => i.json).filter(j => j && j.Event);
const cfg = {
  "Contract Sent":     { nudgeMin: 48, escalateAt: 120 },
  "Payment Pending":   { nudgeMin: 48, escalateAt: 96 },
  "Payment Confirmed": { nudgeMin: 48, escalateAt: 96 },
  "Kickoff Pending":   { nudgeMin: 48, escalateAt: 96 },
};
const out = [];
for (const r of records) {
  const status = r["Status"];
  const c = cfg[status];
  if (!c) continue;
  const lastChange = r["Last Status Change"];
  let hoursElapsed = null;
  if (lastChange) {
    hoursElapsed = (now.getTime() - new Date(lastChange).getTime()) / 3600000;
  }
  const clientName = String(r["Client Name"] || "").trim();
  const already = logs.some(l =>
    String(l.Event || "").indexOf("Nudge Sent") !== -1 &&
    String(l.Event || "").indexOf(status) !== -1 &&
    clientName.length > 0 && String(l.Note || "").indexOf(clientName) !== -1
  );
  let action = "none";
  if (hoursElapsed !== null) {
    if (hoursElapsed >= c.escalateAt) action = "escalate";
    else if (hoursElapsed >= c.nudgeMin && hoursElapsed < c.escalateAt) action = already ? "none" : "nudge";
  }
  out.push({ json: { ...r, hoursElapsed: hoursElapsed === null ? null : Math.round(hoursElapsed * 10) / 10, nudgeAlreadySent: already, action } });
}
return out;
```

**Why this works even when Search Activity Log returns zero items:** the Code node reads from `$('Search Onboarding')` and `$('Search Activity Log')` by explicit name, pulling each node's full output directly rather than depending on whatever happens to flow through a single default input connection. An empty result from Search Activity Log simply becomes an empty array inside the code, which correctly means "nobody has been nudged yet," rather than halting the workflow or blocking Search Onboarding's separate output from being processed.

**Why the dedup check needs three conditions together, not just one:** checking only for the word "Nudge Sent" anywhere in the log would treat any client's nudge as proof that every client has been nudged. Adding the status check narrows it to the right stage. Adding the client name check inside Note narrows it to the right client. All three together are what make it safe to run this check across every client in one pass without cross-contamination between records.

- **Routing:** a Switch node (Route by Status) splits records into four branches by Status. Inside each branch, a second Switch node splits again by the Compute node's `action` field, into nudge, escalate, or nothing.

- **Nudge path (per stage):** send client email, post internal Slack notification, write an Activity Log entry with Event containing both "Nudge Sent" and the stage name, and Note containing `Client: [name] | Stage: [stage] | hoursElapsed: [number]`. This log entry is exactly what the next run's dedup check depends on to avoid a repeat nudge every two hours.

- **Escalate path (per stage):** update Status to `Blocked – [Stage] Timeout`, log the escalation, post an internal Slack alert clearly marked as an escalation rather than a routine nudge. No client-facing email is sent at this stage, since a stalled record at this point needs a human, not another automated message.

- **Untestable in this environment:** Airtable's Last Status Change field is populated automatically and cannot be manually backdated for testing purposes within the platform's own interface. The nudge and escalate paths were validated through direct code review and logical trace-through rather than a live end-to-end test against an artificially aged record. If revisiting this project, manually inserting a test record through the API with a backdated timestamp would allow a full live test of this workflow.

---

## Cross-cutting patterns worth remembering

- **The create-then-log pattern:** almost every workflow ends a successful action by updating the Onboarding record's Status, then immediately writing a matching Activity Log entry. This pair is what makes both the Sweeper's dedup logic and any future audit of "what happened and when" possible.
- **Search-then-branch pattern:** any time one system needs to find a record based on data from another system (a webhook payload, a form submission), the same shape is used: Search node, then an IF node checking whether anything was found, with a Slack alert on the not-found branch rather than letting a silent failure happen.
- **Continue (using error output), not Stop Workflow:** every external API call in this system uses this setting specifically so a single failure does not take down the rest of a client's automated pipeline, and so each failure can be routed to its own specific, informative error handling rather than a generic crash.
