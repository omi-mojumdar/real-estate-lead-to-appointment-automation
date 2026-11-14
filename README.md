# Real Estate Lead-to-Appointment Automation System

An n8n-based automation system for US residential real estate teams. It captures property inquiries through Tally, uses Google Gemini to classify and structure lead information, syncs contacts to HubSpot, sends Gmail follow-ups, and processes Calendly bookings with Gmail and Slack notifications.

> **Portfolio project:** This repository documents a working automation demonstration. It is not a turnkey production deployment. Each brokerage will need its own credentials, CRM configuration, lead fields, messaging rules, and testing.

## Repository description

n8n automation for US residential real estate teams: capture Tally inquiries, qualify leads with Google Gemini, sync contacts to HubSpot, automate Gmail follow-ups, and process Calendly appointments with Gmail and Slack notifications.

## Business problem

Residential real estate teams can lose opportunities when inquiries arrive through disconnected channels, lead details are entered manually, follow-ups are delayed, or appointments are not reflected in the CRM.

This system connects the main steps of the lead journey in one automated process:

**Capture → Validate → Qualify → Update CRM → Follow up → Book appointment**

## What the system does

- Receives property inquiries from a Tally form through an n8n webhook.
- Validates and normalizes submitted contact details.
- Uses Google Gemini to classify the inquiry as **buyer**, **seller**, or **renter**, and extract the prospect's stated requirements.
- Creates or updates a HubSpot contact using the email address as the deduplication key.
- Stores real-estate-specific details in HubSpot custom contact properties.
- Sends lead notifications to the agent through Gmail and a Slack channel.
- Sends an immediate acknowledgment email and schedules later follow-up messages.
- Receives Calendly booking events, sends a confirmation email, alerts the agent in Slack, and updates the HubSpot lead status.

## Architecture

```text
Tally inquiry form
       |
       v
n8n Webhook
       |
       v
Validate and normalize fields
       |
       v
Google Gemini: classify lead + extract requirements
       |
       v
Parse structured AI output
       |
       v
HubSpot: create/update contact
       |                       \
       v                        v
Gmail agent alert          Slack agent alert
       |
       v
Gmail Day 0 acknowledgment
       |
       v
Wait 2 days -> Check HubSpot lead status
       |
       +-- Still eligible --> Gmail Day 2 follow-up
       |
       v
Wait 2 more days -> Check HubSpot lead status again
       |
       +-- Still eligible --> Gmail Day 4 follow-up + Calendly link

Calendly invitee.created
       |
       v
n8n booking workflow
       |
       +--> Gmail booking confirmation
       +--> Slack booking alert
       +--> HubSpot lead status update to CONNECTED
```

## Workflow details

### Workflow 1 — Lead capture, qualification, CRM update, and follow-up

1. **Lead Capture (Webhook):** receives the Tally form submission.
2. **Validate Fields (Code):** extracts Tally's array-based field data, maps it to clean variables, and checks required name and email fields.
3. **AI Classifier (Google Gemini):** classifies the inquiry as buyer, seller, or renter and extracts the lead's stated property requirements.
4. **Extract Requirements (Code):** parses the model response into structured JSON and handles Markdown code fences.
5. **HubSpot — Save Lead:** creates or updates the contact using email as the deduplication key and maps the real-estate custom properties.
6. **Agent notifications:** sends the lead details through Gmail and posts a message in the configured Slack channel.
7. **Day 0 acknowledgment:** sends an immediate email confirming that the inquiry was received.
8. **Day 2 follow-up:** after a two-day wait, checks the lead's HubSpot status before sending the next message.
9. **Day 4 follow-up:** after another two-day wait, checks the status again and sends a final reminder containing the Calendly booking link when the lead remains eligible for follow-up.

### Workflow 2 — Appointment booking

1. **Calendly Trigger:** listens for the `invitee.created` booking event.
2. **Booking confirmation:** sends the prospect a Gmail confirmation using the booking details, including available reschedule and cancellation links from the Calendly payload.
3. **Agent notification:** posts the new appointment details to Slack.
4. **CRM update:** updates the corresponding HubSpot contact's lead status to `CONNECTED`.

## Technology stack

| Tool | Role |
|---|---|
| n8n | Workflow orchestration, webhooks, logic, waits, and integrations |
| Tally | Property inquiry form and lead intake |
| Google Gemini | Lead classification and requirement extraction |
| HubSpot CRM | Contact records, custom properties, and lead status |
| Gmail | Agent notifications, prospect acknowledgments, follow-ups, and booking confirmations |
| Slack | Real-time agent and team notifications |
| Calendly | Appointment scheduling and booking event trigger |

## HubSpot custom contact properties

The project uses the following custom properties:

| Label | Internal name | Type | Example values |
|---|---|---|---|
| Lead Type | `lead_type` | Dropdown | `buyer`, `seller`, `renter` |
| Budget Range | `budget_range` | Single-line text | Prospect-provided budget range |
| Preferred Area | `preferred_area` | Single-line text | Prospect-provided location preference |

Confirm that these properties exist in the target HubSpot account and that their internal names match the field mappings before activating the workflows.

## Lead data handled

The documented workflow normalizes the following fields:

- `firstName`
- `lastName`
- `email`
- `phone`
- `leadType`
- `budget`
- `area`

The AI should classify and structure only information supplied by the prospect. It should not infer protected or sensitive personal characteristics or make housing eligibility decisions.

## Demo resources

- **Workflow demonstration video:** [Watch on Google Drive](https://drive.google.com/file/d/1zhtzaz27QNRyiLj3R80xjmwn63vKcMNr/view?usp=drive_link)
- **n8n workflow export (JSON):** [Open on Google Drive](https://drive.google.com/file/d/1Nkqq_6P6LOIjJbn-ai9ogGMuAXojWL_W/view?usp=drive_link)

The workflow export is hosted externally and is not currently committed to this repository. Add the sanitized JSON export to the repository when you are ready to publish it. Remove credential references, webhook secrets, personal data, and account-specific identifiers first.

## Screenshot

<img width="1870" height="798" alt="8cdb9077-58f2-4d3e-ba89-cc3da0569ad7" src="https://github.com/user-attachments/assets/1f0f959b-709d-479c-9e80-5d587eab02b6" />

## Setup overview

This is a configuration overview, not a one-click installer. Exact node settings may vary with the versions of n8n and each connected service.

1. **Prepare the services:** create or select the Tally form, Gemini API access, HubSpot portal, Gmail account, Slack workspace/channel, and Calendly event type.
2. **Import the workflows:** download the n8n JSON export from the demo resource above and import it into your n8n instance.
3. **Configure credentials:** add the required credentials in n8n for Gemini, HubSpot, Gmail, Slack, and Calendly as applicable. Do not put secrets directly in exported workflow JSON.
4. **Configure Tally:** map the form fields to the values expected by the validation Code node and point the form integration to the n8n production webhook URL.
5. **Configure HubSpot:** create the three custom contact properties listed above and verify the contact and lead-status mappings.
6. **Configure notifications:** set the agent recipient, Slack channel, sender details, and client-approved email templates.
7. **Configure Calendly:** register the n8n webhook for the `invitee.created` event and confirm the event payload includes the contact email and booking URLs required by the workflow.
8. **Test before activation:** run the checklist below with test records and confirm that every branch behaves as expected.

## Validation checklist before client deployment

- [ ] Valid Tally submission creates or updates the intended HubSpot contact.
- [ ] Missing name or email is rejected or routed for review.
- [ ] Duplicate submissions are handled without creating unintended duplicate contacts.
- [ ] Gemini output is valid, parsed JSON and classification is reviewed for accuracy.
- [ ] HubSpot custom properties receive the correct values.
- [ ] Agent Gmail and Slack notifications contain the intended lead details.
- [ ] Day 0, Day 2, and Day 4 messages use the correct recipient and approved content.
- [ ] HubSpot status is rechecked before each scheduled follow-up.
- [ ] A Calendly booking triggers one confirmation, one agent alert, and the intended CRM update.
- [ ] Rescheduling and cancellation behavior is tested.
- [ ] API errors, expired credentials, malformed payloads, and missing fields have defined recovery or human-review paths.
- [ ] Consent, unsubscribe/opt-out handling, data retention, and access permissions are configured for the client's requirements.

## Important production notes

- **Reply detection:** the documented follow-up workflow checks HubSpot lead status. To reliably stop a sequence when a prospect replies by email, implement and test a reply-detection/synchronization path that updates the corresponding HubSpot record before the next follow-up check. Do not assume a reply automatically changes the CRM status unless that integration is configured.
- **Opt-outs:** configure an explicit unsubscribe/opt-out check and suppress further marketing follow-ups after an opt-out. The detailed node breakdown for this demonstration does not document a dedicated opt-out node.
- **Booking changes:** test rescheduling and cancellation events separately; the documented booking workflow describes the new-booking event `invitee.created`.
- **Human oversight:** agents should review uncertain AI classifications and important communications. AI output should not determine whether someone is eligible for housing.
- **Privacy and security:** use fictional sample records in public demos, collect only necessary information, restrict access, and keep API keys, OAuth tokens, webhook secrets, and customer data out of Git history.
- **Client-specific setup:** confirm CRM fields, lead sources, account permissions, messaging consent, and workflow ownership before promising integrations or response-time guarantees.

## Project status

**Portfolio demonstration completed:** November 14, 2026.

The project demonstrates an end-to-end lead intake, AI qualification, CRM synchronization, scheduled email follow-up, and appointment-booking process. Production deployment requires client-specific configuration, security review, and end-to-end testing.

## Author

**Omi Mojumdar** AI Automation & Agent Developer  
[LinkedIn](https://www.linkedin.com/in/omi-mojumdar-900923388) 
· [Email](mailto:omimojumdar88@gmail.com)

---

*Built with n8n to demonstrate practical AI-powered business process automation for US residential real estate teams.*
