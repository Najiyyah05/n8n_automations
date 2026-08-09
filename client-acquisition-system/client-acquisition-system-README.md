# Client Acquisition & Booking Automation System

Coaches and consultants lose potential clients when leads don't get fast,
consistent follow-up. This system automates the entire journey from lead
capture to booked call to closed client — with zero manual admin work.

## What it does

- **Lead Capture** — branded intake form with duplicate detection;
  returning leads update their record instead of creating duplicates
- **Automated Follow-Up** — nudges leads who haven't booked within a
  set window, with escalating touchpoints
- **Booking Sync** — real-time integration with Cal.com; confirmed
  bookings automatically update the CRM and notify the business owner
- **Activity Logging** — full audit trail of every lead interaction,
  for complete pipeline visibility
- **Pipeline Tracking** — leads move through New Lead → Contacted →
  Booked → Closed, with a live reporting view

## Stack

- **n8n** — automation logic and orchestration
- **Airtable** — CRM database + activity log
- **Cal.com** — booking and calendar management
- **Gmail** — automated email sequences

## Workflows

| File | Purpose |
|---|---|
| `1-lead-capture-intake.json` | Form intake, duplicate check, welcome email |
| `2-follow-up-sequence.json` | Scheduled nudges for non-booked leads |
| `3-booking-confirmation.json` | Cal.com webhook → CRM update + notification |
| `4-mark-as-closed.json` | Manual close-out trigger + logging |

## Screenshots

*(embed a few key images here)*

## Notes

Built as a reusable, white-label system — deployable for any coaching or
consulting business with minor customization. Sample data used throughout
is entirely fictional.
