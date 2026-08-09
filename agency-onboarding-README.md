# Agency Client Onboarding Pipeline

A fully automated, error-resistant onboarding system for agencies, built to take a client from
first inquiry to fully onboarded with zero manual setup work, while handling real-world edge
cases like returning clients, multiple simultaneous services, and API failures.

Originally scoped from a real Upwork job post requesting a single, focused onboarding pipeline.
Built out further than the original brief to include dynamic service-based branching, round-robin
team assignment, and dedicated error handling, since these reflect how a real agency's onboarding
actually needs to work.

## The problem

When an agency lands a new client, onboarding involves a lot of scattered manual setup: someone
has to create a CRM record, notify the team, set up a project in the PM tool, create the client's
folder structure, send a welcome email, and assign the right team members their first tasks.
Done manually, this means inconsistency, delays between "client signs" and "team starts working,"
and steps that get silently missed.

## The solution

Three linked n8n workflows that together handle the full journey: lead intake, the booking-to-
onboarding chain, and centralized error handling.

---

## Workflow 1: Lead Intake

**Trigger:** Form submission (name, email, phone, services requested via checkbox: Design and/or
Development)

**Core logic — three possible outcomes per submission:**

1. **No existing record found (brand new lead)**
   → Create a new Clients record, Status = "New Lead", Services Requested = current selection

2. **Existing record found, Status = "Onboarded"**
   → This is a returning client starting a genuinely new engagement. A new Clients record is
   created (fresh project), but their existing Drive Folder ID is carried over, since the client's
   top-level folder should persist across all their engagements, not be recreated each time.

3. **Existing record found, Status = "New Lead" or "Booked" (still mid-pipeline)**
   → The client hasn't been onboarded yet, so this is treated as an update to their current,
   unfinished request. Services Requested is replaced (not merged) with their latest selection,
   since a second submission before onboarding most likely reflects a change of mind, not an
   addition. Status and Drive Folder ID are left untouched.

Each branch logs its own Activity Log entry (`New Lead Created`, `New Engagement Started`, or
`Request Updated`) and sends a tailored confirmation email containing the Cal.com booking link.

**Why three branches instead of a simple create-or-update:** an early version of this workflow
used a straightforward duplicate check, but testing surfaced two real problems: (1) merging
services on every resubmission incorrectly assumed the client wanted to add to their request
rather than replace it, and (2) treating every returning client the same way regardless of
whether they'd already completed a full engagement would have caused an already-onboarded
client's second, unrelated project to overwrite their original record. The three-branch structure
resolves both issues.

---

## Workflow 2: Client Booked → Onboarding Chain

**Trigger:** Webhook (Cal.com "Booking Created" event)

**Step-by-step:**

1. **Find the correct client record.** Cal.com's webhook only provides the booker's email, and
   since a client can have multiple records over time (one per engagement), the search filters by
   both email AND `Status = "New Lead"`, targeting the one engagement currently awaiting a
   booking.
   **Known limitation:** if a client somehow has two simultaneous unbooked engagements, this
   match could resolve to the wrong one. A production-grade fix would pass a unique record
   identifier through the booking link itself (via Cal.com's URL metadata parameters) rather than
   relying on email + status matching.

2. **Update Status → "Booked"**, log the booking date/time.

3. **Create the Asana project**, named after the client, and store the returned project ID back
   in Airtable.

4. **Assign the Account Manager** (always, regardless of services selected): searches the Team
   table for anyone with the Account Manager role, picks whoever currently has the lowest
   `Active Projects` count (round-robin by workload), creates their Asana task, increments their
   count, and logs the assignment.

5. **Resolve the client's Drive folder.** If the client already has a Drive Folder ID stored
   (returning client), it's reused. If not, a new top-level folder is created and the ID is saved
   back to their record.

6. **Loop over each selected service** (Design and/or Development) using n8n's Loop Over Items
   node rather than parallel branching:
   - Finds or creates a *persistent* service-level folder (e.g. "Design") inside the client's
     folder, reused across every engagement of that type for that client.
   - Creates a uniquely-named subfolder for this specific engagement, `[Service] - [Asana Project
     ID]`, so multiple engagements of the same service type never collide.
   - Assigns the relevant team member (Designer or Developer) via the same round-robin logic used
     for the Account Manager.
   - Logs each folder and task-assignment action.

7. **Once the loop completes**, sends the final onboarding email, then updates Status →
   "Onboarded".

**Why a loop instead of parallel IF branching:** the original build split services into separate
items (via a Split Out node) and let them process in parallel, since a client could select one or
both services. This repeatedly caused "multiple matching items" errors once the logic involved
branching (search-then-create-if-needed) and reconverging, because n8n couldn't reliably trace
which of the two parallel items a given expression referred to. Restructuring this as a
sequential loop (one service fully processed before the next begins) eliminates the ambiguity
entirely, since only one item is ever active in that part of the chain at any given moment.

**Resilience:** Asana, Google Drive, and Gmail nodes have Retry On Fail enabled, so brief,
temporary API issues resolve automatically without any manual intervention. Any failure that
exhausts its retries triggers Workflow 3.

---

## Workflow 3: Error Handler

**Trigger:** Error Trigger (n8n's built-in mechanism, fires automatically when a linked workflow
fails)

**Action:** Sends a Slack message to `#automation-alerts` containing the workflow name, the
specific node that failed, the error message, and a timestamp.

**Linked to:** Workflow 1 and Workflow 2, via each workflow's Settings → Error Workflow.

This is the system's safety net: retry logic handles the common case (a brief API hiccup), and
this workflow ensures that anything more serious is surfaced immediately rather than failing
silently, which was an explicit requirement in the original job post this project was inspired
by.

---

## Tools used

| Tool | Purpose |
|---|---|
| n8n | Automation logic and orchestration across all 3 workflows |
| Airtable | Clients table, Team table (round-robin assignment), Activity Log |
| Asana | Project and task creation, team assignment |
| Google Drive | Client and service-level folder structure |
| Cal.com | Booking calendar |
| Gmail | Client-facing email automation |
| Slack | Internal error alerts |

## Airtable structure

**Clients:** Name, Email, Phone, Services Requested (multi-select), Status (New Lead / Booked /
Onboarded), Date Captured, Booking Date/Time, Asana Project ID, Drive Folder ID

**Team:** Name, Role (Account Manager / Designer / Developer), Email, Active Projects (drives
round-robin assignment)

**Activity Log:** Client (linked record), Activity Type, Details, Timestamp

## Workflows included

| File | Purpose |
|---|---|
| `1-lead-intake.json` | Form intake, three-branch duplicate/re-engagement handling, confirmation email |
| `2-client-booked-onboarding-chain.json` | Project creation, folder structure, round-robin team assignment, onboarding completion |
| `3-error-handler.json` | Centralized error catching and Slack alerting |

## Known limitations and possible extensions

- Booking-to-record matching relies on email + status rather than a unique identifier passed
  through the booking link, noted above, this is the main thing worth hardening for real
  production use.
- Team assignment currently supports three fixed roles, could be extended to support
  configurable roles per agency.
- No shared, cross-client "all Design work" view exists by design, folders are organized
  per-client first. A cross-client view would require either duplicating files or a separate
  reporting layer, and was intentionally left out of this version.

## Notes

Built as a self-initiated portfolio project, inspired by a real Upwork job post but extended
beyond its original scope to demonstrate branching logic, workload-based assignment, and proper
error handling. Sample data used throughout is entirely fictional.
