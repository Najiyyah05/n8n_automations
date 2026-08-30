# Automated Client Onboarding System (v2)

An end to end automation that takes a client from first contact through a fully provisioned account, a kicked off project, and a closed final payment, with built in exception handling and follow up throughout. Built in n8n, using Airtable as the system of record.

This is a working prototype, not a production system, but every workflow described here is live, published, and has been tested end to end, including both payment methods it supports.

This version replaces an earlier design that was reworked directly from feedback given by the agency after a live presentation. The section below on what changed and why is kept deliberately, since the reasoning behind each change is as much a part of this project as the final result.

## The problem this solves

Client onboarding usually falls apart in the gaps between tools. A signed contract sits in one place, payment happens somewhere else, project details live in a form nobody checks, and internal setup work depends on someone remembering to do it by hand. Deals stall in these gaps, often without anyone noticing until the client asks what is going on.

This system removes the gaps, while keeping a human in the loop at the two points where judgement genuinely matters: confirming price, and approving what gets billed. Everything else, generating documents, creating accounts, sending communications, tracking who has gone quiet, runs on its own.

## What changed from the first version, and why

The first version of this system triggered off a CRM stage change, collected intake information after payment, and sent a single full payment automatically the moment a contract was signed. After presenting that version to the agency, three changes were requested, and a fourth issue surfaced during rebuilding that was fixed along the way.

**Intake now starts the process, not a CRM stage change.** A client filling out an intake form is what creates their record and kicks off everything downstream. This removes a manual step that used to sit in front of the whole system, and gives the client a clear, obvious first action to take.

**Contract now comes after intake, not before.** Asking a client to hand over business details before anything has been signed asks for effort from someone who has not yet committed to anything. Intake first, contract second, respects that ordering.

**Payment now goes through a human confirmation step before anything is sent to the client**, and is split into two halves rather than one full payment upfront. A deposit is requested once the contract is signed, and the remaining balance is requested once the project is marked complete. Both amounts are confirmed by a team member, in Slack, before either payment link goes out. This was a deliberate design conversation: the actual point where the client and the agency agree on price is the signed contract itself, so this confirmation step is not re negotiating price, it is a safety check that the split was calculated correctly and matched to the right client, before money related communication leaves the building.

**Both payment methods are supported without duplicating logic.** Rather than building a separate workflow for card payments and a separate one for cash, both methods branch out from one shared request step and merge back into one shared confirmation step. Cash confirmation does not involve anyone opening Airtable directly. A client who pays by cash is given a reference number by email, the same identifier already generated for their signed contract, and confirms their own payment through a short form by entering that number. If the number does not match any record, the client is told to re-enter it rather than the submission silently failing.

**A gap was caught and fixed during the rebuild.** Provisioning, the step that creates a client's Drive folder, Asana project, and internal setup, was originally only triggered by a card payment webhook. Once cash payments were added as a separate path, it became clear that a client paying by cash would never reach provisioning at all, since nothing was triggering it on that branch. The fix was merging the card and cash confirmation workflows so both converge into one shared provisioning step, rather than each needing its own copy of that logic.

## Architecture at a glance

**System of record:** Airtable

- **Onboarding** — one record per client, created the moment intake is submitted. This is the spine of the system. Every workflow reads from and writes to this table.
- **Activity Log** — a timestamped history of every event that happens to an Onboarding record, linked to Onboarding. This gives the system a memory of what has already happened, which is what makes duplicate prevention and follow up possible.
- **Intake Data** — the client's own submitted information, captured at the very first step.

**Orchestration:** n8n, nine separate published workflows, each focused on one stage or one job, rather than a single workflow trying to do everything.

**Other tools:** PandaDoc (contracts and e-signature), Paddle (card payments, sandbox, dynamic per client pricing), Google Drive, Asana, Slack (internal team communication and approvals), Gmail (all client facing communication), Cal.com (kickoff scheduling).

A note on this version's communication choices: Slack is used only for internal, team facing messages. Every client facing message goes through Gmail instead, since Slack, WhatsApp, or Telegram are not how this agency's clients actually reach out, and a system should follow how people really communicate, not force a channel onto them because it is convenient to build.

## The onboarding pipeline

```
Intake Received → Price Confirmed → Contract Sent → Contract Signed →
Deposit Awaiting Payment → Deposit Confirmed → Provisioned →
Kickoff Pending → Kickoff Scheduled → Project In Progress →
Project Complete → Final Payment Awaiting Payment → Closed
```

Any stage can also move to a `Blocked – [reason]` state, which pauses automated progress until a human resolves it.

## The nine workflows

### 1. Onboarding Intake Automation
Triggers when a client submits the intake form. Checks for a duplicate submission by matching contact email against existing records. Creates the Onboarding record, then sends a Slack message to the team asking for the agreed price, using n8n's Human in the Loop node so a person can reply with a number directly in Slack rather than needing to open Airtable. The confirmed price is written back automatically, and status moves to Price Confirmed.

### 2. Contract Generation & Sending
Triggers once a price has been confirmed. Generates a contract from a template with the client's exact price filled in, and sends it for signature through PandaDoc. Uses a polling loop to handle PandaDoc's asynchronous document processing, since the API confirms the document was received before it is actually ready to send.

### 3. Contract Signing, Price Confirmation & Payment Request
Triggers on PandaDoc's signature webhook. Matches the signed document back to the correct client, uploads the signed contract to that client's Google Drive folder, and calculates a deposit as half of the agreed price. A second Human in the Loop Slack message asks the team to confirm this deposit amount before anything is sent. Once confirmed, the client is either sent a Paddle payment link by email, or, if paying by cash, is emailed their unique reference number instead.

### 4. Cash Payment Confirmation (Deposit)
A standalone form a client uses to confirm they have paid by cash. They enter the reference number they were emailed, which is checked against the Onboarding table. A match updates their record and logs the payment. No match returns a message asking them to re-enter the number, rather than failing silently.

### 5. Payment Confirmed → Provisioning
Triggers from either a Paddle webhook confirming a card payment, or the cash confirmation form. Both paths independently confirm the deposit was paid, then merge into one shared provisioning step: creating an Asana project for the client, since their Drive folder was already created in step 3. This merge exists specifically because a card only and a cash only version of this step would otherwise duplicate the same provisioning logic twice, once per payment method.

### 6. Welcome Email & Kickoff Scheduling
Two independent triggers in one workflow. The first fires once provisioning is complete and sends a welcome email with a Cal.com link. The second listens for a Cal.com booking confirmation, whenever it happens, and updates the client's status.

### 7. Project Completed & Final Payment Request
Triggers from a form a project lead uses to mark a client's project as complete, since no connected tool can automatically detect that real project work has finished. Calculates the remaining balance, confirms it with the team through the same Human in the Loop pattern used for the deposit, then requests final payment through whichever method the client is using.

### 8. Final Payment Confirmation & Closeout
Mirrors workflow 5's structure: a Paddle webhook and a second cash confirmation form both converge on the same final step, marking the client's Onboarding record as Closed.

### 9. The Sweeper
Runs every two hours. Checks every record sitting in a stage that depends on someone else taking action: the team confirming a price, a client signing a contract, or a client completing either payment. Sends one reminder after a set number of hours with no movement, using Gmail for anything client facing and Slack for anything internal, and escalates to the team if there is still no progress after a longer threshold, without repeating the same reminder endlessly.

## Exception handling

- **A dedicated Global Error Handler workflow** catches any failure not otherwise handled inside a workflow, formats the error, and posts it to Slack. Every other workflow points to this one as its error workflow, so failures are reported consistently in one place rather than needing a Slack alert node manually wired after every single step.
- **Human in the Loop confirmation, used twice**, once for initial pricing, once for each payment amount, exists specifically so a person reviews numbers and client attribution before anything financial goes out, without needing that person to touch Airtable directly.
- **Matching between systems uses stored identifiers, not names.** The cash payment confirmation form matches on a reference number, not a typed client name, so a mistyped name can never link a payment to the wrong client. A wrong or mistyped identifier fails safely into a re-entry prompt, rather than either rejecting silently or, worse, matching to the wrong record.
- **The Sweeper distinguishes internal stalls from client stalls.** A team member forgetting to confirm a price is treated differently, and on a different schedule, than a client who has not yet paid, since the appropriate nudge and audience differ for each.

## A real lesson from testing

During an earlier live demo, an Airtable trigger was left active after the presentation ended, using a workaround formula field instead of Airtable's native trigger option. That formula field was also being updated by the workflow's own actions, which created a loop: the trigger saw a change, ran the workflow, the workflow updated the same field, the trigger saw that as a new change, and ran again. The same test record was onboarded, and issued a new contract, well over a hundred times before it was caught.

The fix was twofold: never trigger a workflow off a field that workflow itself writes to without also checking something that only becomes true once, and add a duplicate action guard to every workflow that sends something externally, not only to the ones that create new records. The first version of this system had that guard on record creation, but not on contract sending, which is why nothing stopped the repeat sends once the loop started.

## Stack

Airtable, n8n, PandaDoc, Paddle, Google Drive, Asana, Slack, Gmail, Cal.com.
