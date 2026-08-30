# Architecture and Technical Notes (v2)

This is the detailed companion to the README, covering the revised system built after agency feedback. It exists so that months from now, without memory of building this, you can open this file and reconstruct exactly how each piece works, not just what it does.

This document assumes familiarity with the general patterns already established in the first version: the create-then-log pairing, search-then-branch matching, and "Continue (using error output)" on external calls. Those patterns are unchanged here. What follows focuses on what is new or different in this version.

---

## Data model changes

### Onboarding table
No longer created from a linked Deal. Created directly from an Intake Data submission. Key fields, in addition to everything carried over from the first version:

- **Deal Value** — no longer a lookup from a linked Deal, now written directly by the pricing workflow after a team member confirms it through Slack.
- **Deposit Amount** — a formula field, half of Deal Value. Read directly by workflow 3's Human in the Loop confirmation step, no Code node needed to calculate it.
- **Remaining Balance** — a formula field, Deal Value minus Deposit Amount. Read directly by workflow 7's confirmation step, same reasoning.
- **Final Payment Amount** — written after the team confirms the remaining balance figure in workflow 7.
- **Payment Method** — single select, E-payment or Cash. Drives every Switch node branching decision from workflow 3 onward.
- **PandaDoc Document ID** — same field as the first version, but now doing double duty. It is still used to match the signature webhook back to the correct record, and it is also the reference number emailed to clients paying by cash, since it is already a unique identifier generated per client with no extra field needed.

### Intake Data table
Now the actual entry point of the whole system, not a mid-process form. Fields cover only what is needed to create a record and generate a contract; deeper operational detail (branding assets, tools used, team members) is deliberately not collected here, since asking for that before a contract exists is unnecessary friction.

---

## Workflow 1: Onboarding Intake Automation

- **Trigger:** Airtable Trigger on Intake Data, record created.
- **Duplicate check:** Airtable Search on Onboarding:
```
{Contact Email} = "{{ $json.fields['Contact Email'] }}"
```
- **On no match:** create the Onboarding record, Status set to Intake Received, then log to Activity Log.
- **Pricing step:** a Slack node using n8n's `sendAndWait` operation, which is the mechanism behind the Human in the Loop pattern used throughout this version. The message includes Client Name and enough project context for someone to give an informed number, and the node pauses workflow execution until a reply is captured.
- **On response:** an Airtable Update node writes the confirmed number into Deal Value, and Status moves to Price Confirmed. This status change is what triggers workflow 2.
- **Every step from Search onward has a paired error branch**, alerting the team on Slack if the duplicate check, record creation, pricing prompt, or the write-back fails.

---

## Workflow 2: Contract Generation & Sending

Structurally unchanged from the first version's contract workflow. The only functional difference: the trigger condition is now Status equals Price Confirmed instead of Deal Won, and Deal Value is read directly off the Onboarding record rather than through a linked Deal lookup. The PandaDoc polling loop, the token mapping, and the native Google Docs template (needed since PDF templates do not reliably accept tokens) all carry over unchanged.

---

## Workflow 3: Contract Signing, Price Confirmation & Payment Request

- **Trigger:** Webhook, POST, Authentication None, same reasoning as the first version's PandaDoc webhook.
- **Match logic:** unchanged, matching on PandaDoc Document ID.
- **New step, Drive upload:** once matched and marked Contract Signed, a node group creates or locates the client's Drive folder, fetches the signed document via an HTTP request to PandaDoc, uploads the file, and updates the Onboarding record with a link to it. This exists so a signed copy of every contract is retrievable without needing to go back into PandaDoc.
- **Deposit confirmation:** a second `sendAndWait` Slack node, reading Deposit Amount directly from the formula field on the Onboarding record. The message shown to the team includes total Deal Value alongside the calculated 50 percent figure, so the confirmation is checking the split and the client match, not re-litigating the agreed price, which was already settled at signature.
- **Payment Method branch:**
  - **E-payment:** creates a Paddle transaction using the same non-catalog dynamic pricing pattern from the first version, this time using the confirmed Deposit Amount rather than a full deal value. Emails the client a payment link via Gmail.
  - **Cash:** emails the client their PandaDoc Document ID as a reference number to use when confirming payment, and posts an internal Slack alert that a cash payment is expected.
- Both branches log to Activity Log, noting the method used.

---

## Workflow 4: Cash Payment Confirmation (Deposit)

- **Trigger:** Form Trigger, a single field asking the client to enter their reference number.
- **Search:** Airtable Search on Onboarding, matching the submitted number against PandaDoc Document ID.
- **IF node:** on no match, the form's own ending message tells the client the number was not recognized and asks them to check and re-enter it. This is handled by the form's built in conditional ending rather than a separate email, since the client is still present in the form at that moment and does not need to wait for a follow-up message.
- **On match:** updates the record (or creates a log entry, depending on which branch of the underlying IF is taken) confirming the deposit was received by cash.
- This workflow originally ended here. It was later merged into workflow 5, described below, once it became clear a cash payment had nowhere automated to go afterward.

---

## Workflow 5: Payment Confirmed → Provisioning

This is the workflow that resulted from merging the original card-only provisioning trigger with the cash confirmation flow above, once the gap was caught.

- **Two inputs converging on one shared path:**
  - **Path A:** Paddle Webhook, POST, Authentication None. Filters on transaction.completed. Matches the transaction back to the Onboarding record.
  - **Path B:** Form Trigger, the same cash confirmation form from workflow 4. Matches the submitted reference number the same way.
- Both paths independently update the record and log "Payment Confirmed," then each runs a small "Prep name and record ID" Set node, standardizing the shape of the data each path produces so the next step does not need to know which path it came from.
- **Merge node**, set to Append mode, with two inputs, one from each path. Whichever path actually ran is the one that reaches the Merge node; the other simply never fires for that execution. This is the mechanism that lets one shared provisioning step serve both payment methods without duplicating the Asana creation logic twice.
- **After the merge:** creates the Asana project for the client. The Drive folder is not created here, since it already exists from workflow 3's contract upload step.
- **On success:** Status to Provisioned, log to Activity Log.
- **On Asana failure:** Status to Blocked (Asana), log, Slack alert.

**Why this merge matters structurally:** without it, a card-paying client and a cash-paying client would need two entirely separate provisioning workflows, identical except for their trigger, which means any future change to provisioning logic would need to be made twice, in two places, with the risk of the two copies drifting out of sync. The Merge node avoids that by making payment method irrelevant past this one point.

---

## Workflow 6: Welcome Email & Kickoff Scheduling

Unchanged from the first version. Two independent triggers, Branch A on Status equals Provisioned sending the welcome email, Branch B listening for a Cal.com booking webhook. Both branches remain functionally identical to the original design.

---

## Workflow 7: Project Completed & Final Payment Request

Mirrors workflow 3's structure closely, but triggered differently and using the remaining balance instead of the deposit.

- **Trigger:** Form Trigger, "Mark Project as Complete," submitted by a project lead, since no connected tool can automatically detect that the actual client work is finished.
- **Match:** Airtable Search on Onboarding by Client Name.
- **On match:** Status to Project Complete, log to Activity Log.
- **Confirmation:** a `sendAndWait` Slack node reading Remaining Balance directly from its formula field, same reasoning as the deposit confirmation, no Code node needed since Airtable already computed the figure.
- **Payment Method branch:** identical structure to workflow 3, Paddle transaction and email for e-payment, reference number email and Slack alert for cash.

---

## Workflow 8: Final Payment Confirmation & Closeout

Structurally identical to workflow 5's convergence pattern: a Paddle webhook path and a cash confirmation form path (a second form, separate from workflow 4's, since this one confirms the final payment rather than the deposit) both feed into the same shared closing logic; matching, updating Status to Closed, and logging the final Activity Log entry.

---

## Workflow 9: Global Error Handler

A standalone workflow: an Error Trigger node, which n8n automatically calls whenever another workflow fails in a way not otherwise caught inside that workflow, a Code node formatting the incoming error data into a readable message, and a Slack node posting it.

Every other workflow in this system has this one selected under its own Settings tab, under Error Workflow. This replaces having a manually wired Slack alert node after every single external call, in favor of one consistent reporting path for anything not explicitly handled elsewhere. Explicit error branches, the ones that also update Status to a specific Blocked reason, are kept in place separately, since the global handler only knows that something failed, not what business-specific status change should follow.

---

## The Sweeper (workflow 10 in numbering, workflow 9 after the merge reduced the total count)

Rebuilt against the new pipeline's stages. Monitors:

- **Intake Received** (internal: the team has not yet confirmed a price) — shorter thresholds than client-facing stages, since this is entirely within the agency's own control
- **Contract Sent** (client-facing: waiting on signature)
- **Deposit Awaiting Payment** (client-facing: waiting on either payment method)
- **Final Payment Awaiting Payment** (client-facing: waiting on either payment method)

Internal-facing nudges route to Slack only. Client-facing nudges route to Gmail for the client and Slack for internal visibility. The same dedup logic from the first version, checking Activity Log for an existing "Nudge Sent" entry matching this client and stage before sending another, carries over unchanged, since that logic was never tied to specific stage names.

---

## A mistake worth remembering in detail

The first version of this system, during a live demo, was left with all workflows active afterward. The Onboarding table's trigger used a formula field, something like Last Modified Time, as a workaround since Airtable's native trigger options did not behave as expected in n8n. That same field was also updated by the workflow's own status-change actions.

The result was a loop: the trigger polled, saw the formula field's value differ from last check, ran the workflow, the workflow changed the record's status, which changed the formula field's value again, the next poll saw another difference, and ran again. Since the record already sat at a status where contract generation would fire, a new contract was generated and sent, and a new Activity Log entry written, on every single one of those repeated runs. The same test record was processed well over a hundred times before this was noticed.

The first version had a duplicate-prevention check on record creation, matching Client Name and Status before creating a new Onboarding record. It had no equivalent check on contract sending itself, so once the loop started, nothing stopped repeated sends against the same, already-existing record.

Two changes came out of this, both applied going forward: avoid triggering a workflow off any field that workflow itself writes to, unless a separate, one-time-only condition is also checked; and add a duplicate-action guard to every workflow step that sends something externally, not only to the workflow steps that create new records. A duplicate record is a visible, obvious mistake. A duplicate external action, a contract re-sent, a payment link re-issued, is a much worse one to make silently, since it looks like the system is working right up until a client asks why they received the same email six times.
