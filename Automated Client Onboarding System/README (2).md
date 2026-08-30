# Automated Client Onboarding System

An end to end automation that takes a client from first contact through a fully provisioned account, a kicked off project, and a closed final payment, with built in exception handling and follow up throughout. Built in n8n, using Airtable as the system of record.

This is a working prototype, not a production system, but every workflow described here is live, published, and has been tested end to end, including both payment methods it supports.

This version was rebuilt directly from feedback given after presenting an earlier design. The section below on what changed and why is kept deliberately, since the reasoning behind each change is as much a part of this project as the final result.

## The problem this solves

Client onboarding usually falls apart in the gaps between tools. A signed contract sits in one place, payment happens somewhere else, project details live in a form nobody checks, and internal setup work depends on someone remembering to do it by hand. Deals stall in these gaps, often without anyone noticing until the client asks what is going on.

This system removes the gaps, while keeping a human in the loop at the two points where judgement genuinely matters: confirming price, and approving what gets billed. Everything else, generating documents, creating accounts, sending communications, tracking who has gone quiet, runs on its own.

## What changed from the first version, and why

The first version triggered off a CRM stage change, collected intake information after payment, and sent a single full payment automatically the moment a contract was signed. After presenting that version, three changes were requested, and a fourth issue surfaced during rebuilding that was fixed along the way.

**Intake now starts the process, not a CRM stage change.** A client filling out an intake form is what creates their record and kicks off everything downstream.

**Contract now comes after intake, not before.** Asking a client to hand over business details before anything has been signed asks for effort from someone who has not yet committed to anything.

**Payment now goes through a human confirmation step before anything is sent to the client**, and is split into two halves rather than one full payment upfront. A deposit is requested once the contract is signed, and the remaining balance is requested once the project is marked complete. Both amounts are confirmed by a team member, in Slack, before either payment link goes out. The actual point where the client and the agency agree on price is the signed contract itself, so this confirmation step is not re-negotiating price, it is a safety check that the split was calculated correctly and matched to the right client before money-related communication leaves the building.

**Both payment methods are supported without duplicating logic.** Card and cash payments branch out from one shared request step and merge back into one shared confirmation step. A client paying by cash is emailed a reference number, the same identifier already generated for their signed contract, and confirms their own payment through a short form by entering that number. An incorrect number returns a message asking them to re-enter it, rather than failing silently.

**A gap was caught and fixed during the rebuild.** Provisioning was originally only triggered by a card payment webhook. Once cash payments were added as a separate path, a client paying by cash would never have reached provisioning at all, since nothing on that path triggered it. The fix was merging the card confirmation and cash confirmation into a single workflow, so both payment methods converge on the same shared provisioning step rather than needing two separate copies of that logic. This is the same pattern already used for the final payment stage, where both payment methods have always converged into one shared closing step.

## Architecture at a glance

**System of record:** Airtable

- **Onboarding** — one record per client, created the moment intake is submitted.
- **Activity Log** — a timestamped history of every event on an Onboarding record, linked to it.
- **Intake Data** — the client's own submitted information, captured at the very first step.

**Orchestration:** n8n, nine published workflows.

**Other tools:** PandaDoc, Paddle, Google Drive, Asana, Slack (internal only), Gmail (all client-facing communication), Cal.com.

Slack is used only for internal, team-facing messages in this version. Every client-facing message goes through Gmail instead, since Slack, WhatsApp, or Telegram are not how this agency's clients actually reach out.

## The onboarding pipeline

```
Intake Received → Price Confirmed → Contract Sent → Contract Signed →
Deposit Awaiting Payment → Deposit Confirmed → Provisioned →
Kickoff Pending → Kickoff Scheduled → Project In Progress →
Project Complete → Final Payment Awaiting Payment → Closed
```

Any stage can also move to a `Blocked – [reason]` state, pausing automated progress until a human resolves it.

## The nine workflows

### 1. Onboarding Intake Automation
Triggers when a client submits the intake form. Checks for duplicates by contact email. Creates the Onboarding record, then sends a Slack message asking the team for the agreed price, using n8n's Human in the Loop node so a person replies with a number directly in Slack rather than opening Airtable. The confirmed price is written back automatically, status moves to Price Confirmed.

### 2. Contract Generation & Sending
Triggers once a price has been confirmed. Generates a contract from a template with the client's price filled in, sends it for signature through PandaDoc. Uses a polling loop to handle PandaDoc's asynchronous document processing.

### 3. Contract Signing, Price Confirmation & Payment Request
Triggers on PandaDoc's signature webhook. Matches the signed document back to the correct client, uploads the signed contract to that client's Google Drive folder, and calculates a deposit as half of the agreed price. A second Human in the Loop Slack message asks the team to confirm this deposit amount before anything is sent. The client is then either emailed a Paddle payment link, or, if paying by cash, emailed their unique reference number instead.

### 4. Paddle Payment → Provisioning
This workflow has two independent entry points that converge on one shared path.

- **Card path:** a Paddle webhook confirms payment, matches it to the client's record.
- **Cash path:** a form the client uses to confirm their own cash payment, entering the reference number they were emailed. It is checked against the Onboarding table; a match confirms the payment, no match asks them to re-enter the number.

Both paths independently confirm the deposit and log the event, then merge into one shared provisioning step: creating an Asana project for the client (their Drive folder already exists from step 3). This merge exists specifically so a card-only and a cash-only version of provisioning never need to be built and maintained as two separate copies of the same logic.

### 5. Welcome Email & Kickoff Scheduling
Two independent triggers in one workflow. The first fires once provisioning is complete and sends a welcome email with a Cal.com link. The second listens for a Cal.com booking confirmation and updates the client's status.

### 6. Project Completed & Final Payment Request
Triggers from a form a project lead uses to mark a client's project as complete, since no connected tool can automatically detect that real project work has finished. Calculates the remaining balance, confirms it with the team through the same Human in the Loop pattern used for the deposit, then requests final payment through whichever method the client is using.

### 7. Final Payment Made & Project Closed
Mirrors workflow 4's convergence pattern: a Paddle webhook path and a cash confirmation form path both feed into the same shared closing logic, marking the client's Onboarding record as Closed.

### 8. Global Error Handler
A standalone workflow that catches any failure not otherwise handled inside another workflow, formats it, and posts it to Slack. Every other workflow points to this one as its error workflow, so failures are reported consistently in one place.

### 9. The Sweeper
Runs every two hours. Checks every record sitting in a stage that depends on someone else taking action: the team confirming a price, a client signing a contract, or a client completing either payment. Sends one reminder after a set number of hours with no movement, using Gmail for anything client-facing and Slack for anything internal, and escalates to the team if there is still no progress after a longer threshold.

## Exception handling

- **The Global Error Handler** catches unhandled failures across every workflow, reported consistently in one place instead of a Slack alert node wired manually after every step.
- **Human in the Loop confirmation, used at every point money changes hands**, so a person reviews the amount and the client attribution before anything financial goes out, without needing to touch Airtable directly.
- **Matching between systems uses stored identifiers, not names.** Cash payment confirmation matches on a reference number, not a typed client name, so a mistyped name can never link a payment to the wrong client. A wrong identifier fails safely into a re-entry prompt.
- **The Sweeper distinguishes internal stalls from client stalls**, since a team member forgetting to confirm a price needs a different nudge, and a different audience, than a client who has not yet paid.

## A real lesson from testing

During an earlier live demo, an Airtable trigger was left active afterward, using a workaround formula field instead of Airtable's native trigger option. That formula field was also updated by the workflow's own actions, creating a loop: the trigger saw a change, ran the workflow, the workflow changed the field, the trigger saw that as a new change, and ran again. The same test record was onboarded, and issued a new contract, well over a hundred times before it was caught.

The fix: never trigger a workflow off a field that workflow itself writes to without also checking something that only becomes true once, and add a duplicate-action guard to every workflow step that sends something externally, not only the ones that create new records.

## Stack

Airtable, n8n, PandaDoc, Paddle, Google Drive, Asana, Slack, Gmail, Cal.com.
