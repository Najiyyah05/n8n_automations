# Automated Client Onboarding System

An end to end automation that takes a client from "deal won" through a fully provisioned, kicked off account, with built in exception handling and follow up. Built in n8n, using Airtable as the system of record.

This project was built as a technical assessment. It is a working prototype, not a production system, but every workflow described here is live and has been tested with real data.

## The problem this solves

Client onboarding usually falls apart in the gaps between tools. A contract gets signed in one place, payment happens somewhere else, intake information lives in a form nobody checks, and the internal setup work (folders, project boards, team channels) depends on someone remembering to do it. Deals stall in these gaps, often without anyone noticing until the client asks what is going on.

This system removes the gaps. Every stage of onboarding, from the moment a deal is marked won to the moment a kickoff call is booked, is tracked in one place, moved forward automatically, and monitored for stalls. When something breaks, a human gets alerted with enough context to fix it quickly, rather than discovering the problem days later.

## Architecture at a glance

**System of record:** Airtable, four tables

- **Deals** — the sales pipeline. Pre existing, not part of this build.
- **Onboarding** — one record per client, created only when a Deal reaches "Closed Won." This is the spine of the whole system. Every workflow reads from and writes to this table.
- **Activity Log** — a timestamped history of every event that happens to an Onboarding record. Linked to Onboarding. This is what makes the exception handling possible, since it gives the system a memory of what has already happened.
- **Intake Data** — client submitted business information, collected through an Airtable form after payment is confirmed.

**Orchestration:** n8n, seven separate workflows, one per major trigger event rather than a single monolithic flow. This keeps each workflow focused on one job and much easier to debug in isolation.

**Other tools:** PandaDoc (contracts and e-signature), Paddle (payment, sandbox mode, used as a combined invoice and checkout system), Google Drive, Asana, Slack, Gmail, Cal.com.

## The onboarding pipeline

Status values move in this order:

```
Deal Won → Contract Sent → Contract Signed → Payment Pending → Payment Confirmed →
Intake Received → Provisioning → Provisioned → Kickoff Pending → Kickoff Scheduled →
Onboarding Complete
```

Any stage can also move to a `Blocked – [reason]` state, which pauses automated progress until a human resolves it.

A quick note on ordering: contract signing happens before intake information is requested, not after. Asking a client to hand over operational details such as branding assets and team lists before they have signed anything creates unnecessary friction and asks for effort from someone who has not yet committed. Getting the signature first, matching the terms already agreed during the sales conversation, keeps the sequence aligned with how the client actually experiences it.

## The seven workflows

### 1. Deal Won → Onboarding Record Creation
Triggers when a Deal's stage changes to "Closed Won." Creates the corresponding Onboarding record, with duplicate protection so the same deal cannot create two records if the trigger fires more than once.

### 2. Contract Generation & Send
Generates a contract from a template using PandaDoc's API and sends it for signature. Handles PandaDoc's asynchronous document processing with a polling loop, since the API returns immediately but the document itself takes a few seconds to finish generating on their end.

### 3. Signature Received → Payment Trigger
Listens for a PandaDoc webhook confirming the document has been signed. Matches the incoming event back to the correct Onboarding record, then creates a Paddle transaction using the client's exact deal value, pulled dynamically rather than relying on a fixed price. Emails the client a payment link.

### 4. Payment Confirmed → Intake Trigger
Listens for a Paddle webhook confirming payment. Updates status and sends the client a link to the intake form, so the next step in the process can begin.

### 5. Intake Received → Provisioning
Triggers when a new intake form submission comes in. Matches it back to the correct Onboarding record, then handles internal setup: creates a Google Drive folder, an Asana project, and a Slack channel for the client, inviting the relevant team member automatically. Each of these three steps is tracked and reported on independently, so a failure in one does not get lost inside a generic "something went wrong" message.

### 6. Welcome Email + Kickoff Scheduling
Two independent triggers living in one workflow, since they represent the same stage of the client journey. The first fires once provisioning is complete and sends a welcome email with a scheduling link. The second listens for a Cal.com booking confirmation, whenever it happens, and updates the record accordingly.

### 7. The Sweeper
Runs on a schedule, every two hours. Checks every Onboarding record sitting in a stage that depends on the client taking action (Contract Sent, Payment Pending, Payment Confirmed, Kickoff Pending) and calculates how long it has been sitting there. If a record crosses a set threshold, it sends the client a gentle reminder and logs that the reminder was sent, so it does not get sent again on the next run. If a record crosses a second, longer threshold without any progress, the system stops nudging and escalates instead, flagging the record as blocked and alerting the team internally.

## Exception handling

Nothing in this system assumes the happy path. A few examples of how failure is handled directly rather than left to surface as a confusing error later:

- Every external API call (PandaDoc, Paddle, Google Drive, Asana, Slack, Gmail) uses explicit error branching rather than letting failures pass through silently. A failed step updates the record's status to a specific `Blocked` reason, logs the failure with the actual error message attached, and alerts the team on Slack.
- The system distinguishes between a request that reached a server and got rejected, and a request that never reached a server at all, since these need to be handled differently and n8n does not always route them the same way by default.
- Records that are already blocked are excluded from the Sweeper's monitoring, since a blocked record is a human's responsibility to resolve, not something more automated nudging will fix.
- Matching between systems (PandaDoc documents, Paddle transactions, intake submissions) is done through stored unique identifiers rather than client names, since names can be duplicated, mistyped, or edited. Where a client facing form needed to carry an identifier for matching purposes, that field is hidden from view but still transmitted, with the matching logic designed so a tampered or missing value fails safely into a manual review path rather than silently linking to the wrong client.

## A few honest lessons learned

- PandaDoc's token replacement was unreliable when the source template was a converted PDF, even though the placeholders looked correct visually. Using a native Google Docs template instead resolved it immediately. Worth knowing before assuming a PDF template will behave the same way.
- Paddle requires a verified default checkout domain before any transaction can be created at all, which is not obvious from the transaction API's own documentation and only surfaces as an error once you try.
- Slack's API will reject an attempt to add the channel's own creator to a channel it just created, since that identity is already technically a member. This looks like a bug the first time you see it, but it is intentional behavior once you understand what is actually happening.
- Airtable's form prefill feature silently drops the value of any field that has been manually hidden from the form builder. The correct way to prefill a hidden field is a separate `hide_FieldName=true` URL parameter, which keeps the field's prefilled value intact while hiding it from the person filling out the form.

## What was intentionally left out of scope

This is a prototype built to demonstrate the orchestration logic clearly, not a production deployment. A few things were deliberately simplified:

- Webhook payload signature verification (HMAC validation) was identified during the build as a real production hardening step and was deprioritized given the project timeline, in favor of getting the full pipeline working end to end first.
- The payment checkout page itself renders through Paddle's own hosted domain rather than a custom branded page, since building a full front end checkout experience was outside the scope of an orchestration focused build.
- SLA thresholds are hardcoded directly in the Sweeper's logic rather than pulled from a configuration table. A configuration table would be a reasonable next step if this were extended into an ongoing product rather than a fixed prototype.

## Stack

Airtable, n8n, PandaDoc, Paddle, Google Drive, Asana, Slack, Gmail, Cal.com.
