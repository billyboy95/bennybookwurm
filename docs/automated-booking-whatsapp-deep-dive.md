# Automated Booking Workflow and WhatsApp Integration Deep Dive

This repository is currently a greenfield project with only a README, so this
document defines a practical target architecture rather than auditing an
existing implementation.

## Executive summary

Build the booking system around a clear booking state machine, a small set of
durable domain records, asynchronous automation jobs, and a channel-agnostic
messaging layer. WhatsApp should be treated as one delivery channel, not as the
booking system itself.

Recommended MVP:

1. Customer chooses a service, staff member, location, date, and slot.
2. System creates a provisional hold with an expiry.
3. Customer confirms details and, if required, pays a deposit.
4. Booking becomes confirmed.
5. Automation sends WhatsApp confirmations, reminders, reschedule links, and
   post-booking follow-ups.
6. WhatsApp replies are received through webhooks and routed into booking
   actions or staff review.

## Core product assumptions to confirm

- Business type: salon, clinic, consultation, class, rental, or another model.
- Resource model: one staff member per booking, multiple resources, or pooled
  capacity.
- Booking duration: fixed per service or variable.
- Time zones: single business timezone or per-location timezone.
- Payments: none, deposit-only, or full prepayment.
- Cancellation policy: free cancellation window, fees, no-show handling.
- Customer identity: phone-only, email plus phone, or authenticated accounts.
- Staff workflow: customer self-service only, admin-assisted, or both.
- WhatsApp number ownership: new business number or existing number.
- Regions served: affects language, template content, consent, and privacy.

## Booking lifecycle

### Booking states

Use explicit states instead of deriving status from timestamps or payments.

| State | Meaning | Typical transitions |
| --- | --- | --- |
| `draft` | Customer has started a flow but no slot is held. | `held`, `abandoned` |
| `held` | Slot is temporarily reserved while details/payment complete. | `confirmed`, `expired`, `cancelled` |
| `confirmed` | Customer has an active booking. | `rescheduled`, `cancelled`, `completed`, `no_show` |
| `rescheduled` | Original booking was moved to another time. | Terminal for original booking, new booking is `confirmed` |
| `cancelled` | Customer or business cancelled. | Terminal |
| `expired` | Hold expired before confirmation. | Terminal |
| `completed` | Appointment happened. | Terminal |
| `no_show` | Customer did not attend. | Terminal |

Keep a separate event log so support staff can see how the booking reached its
current state.

### Standard customer flow

1. **Service selection**
   - Customer selects service/category.
   - System calculates duration, price, deposit requirement, buffer time, and
     eligible staff/resources.

2. **Availability search**
   - Query available slots using service duration, staff schedule, existing
     confirmed bookings, active holds, breaks, holidays, and location timezone.
   - Return slots in customer-local display format while storing canonical UTC
     timestamps.

3. **Slot hold**
   - Create a short-lived hold, commonly 5-15 minutes.
   - Holds prevent double booking while the customer enters details or pays.
   - A background job expires holds that are not confirmed.

4. **Customer details and consent**
   - Capture name, phone, optional email, notes, and consent to receive
     WhatsApp transactional messages.
   - Normalize phone numbers to E.164 format.

5. **Payment or deposit**
   - If payment is required, create a payment intent tied to the hold.
   - Confirm the booking only after payment succeeds.
   - If payment fails or expires, leave the hold to expire or explicitly cancel
     it.

6. **Confirmation**
   - Convert hold to confirmed booking in a single transaction.
   - Emit a `booking.confirmed` domain event.
   - Send confirmation via WhatsApp, email, or SMS depending on preference and
     delivery eligibility.

7. **Pre-appointment automation**
   - Send reminders, directions, intake links, preparation instructions, and
     reschedule/cancel links.
   - Schedule jobs relative to the appointment time, not relative to booking
     creation.

8. **Day-of handling**
   - Optional "Reply 1 to confirm" or "Reply R to reschedule" workflow.
   - If customer replies, process through WhatsApp webhook and update booking
     intent/action.

9. **Post-appointment**
   - Mark completed/no-show.
   - Send thank-you message, review request, rebooking prompt, or invoice.

## Suggested domain model

Minimum tables/collections:

| Entity | Key fields |
| --- | --- |
| `customers` | `id`, `name`, `phone_e164`, `email`, `whatsapp_opt_in_at`, `locale`, `timezone` |
| `services` | `id`, `name`, `duration_minutes`, `price`, `deposit_amount`, `buffer_before`, `buffer_after`, `active` |
| `staff` | `id`, `name`, `timezone`, `active` |
| `locations` | `id`, `name`, `timezone`, `address`, `active` |
| `staff_services` | `staff_id`, `service_id` |
| `availability_rules` | `staff_id`, `weekday`, `start_time`, `end_time`, `effective_from`, `effective_to` |
| `availability_exceptions` | `staff_id`, `starts_at`, `ends_at`, `reason` |
| `bookings` | `id`, `customer_id`, `service_id`, `staff_id`, `location_id`, `starts_at`, `ends_at`, `status`, `source`, `created_at` |
| `booking_holds` | `id`, `service_id`, `staff_id`, `starts_at`, `ends_at`, `expires_at`, `customer_id`, `status` |
| `booking_events` | `id`, `booking_id`, `event_type`, `actor_type`, `payload`, `created_at` |
| `payments` | `id`, `booking_id`, `provider`, `provider_ref`, `amount`, `currency`, `status` |
| `message_threads` | `id`, `customer_id`, `channel`, `provider_thread_ref`, `last_inbound_at` |
| `messages` | `id`, `thread_id`, `booking_id`, `direction`, `template_name`, `body`, `provider_message_id`, `status`, `error_code`, `created_at` |
| `automation_jobs` | `id`, `booking_id`, `job_type`, `run_at`, `status`, `attempts`, `last_error` |
| `webhook_events` | `id`, `provider`, `provider_event_id`, `payload_hash`, `processed_at`, `status` |

## Availability and double-booking prevention

Double-booking prevention should happen in the database, not only in
application logic.

Recommended safeguards:

- Store all booking and hold times in UTC.
- Use a transaction when confirming a booking.
- Check for overlapping confirmed bookings and active holds.
- Put a database-level exclusion constraint or equivalent lock around
  `staff_id/resource_id + time range` if the selected database supports it.
- Expire holds with a scheduled job and exclude expired holds from availability.
- Treat rescheduling as creating a new confirmed slot and cancelling or marking
  the old booking as rescheduled in one transaction.

## Automation design

Use domain events to decouple booking changes from messaging.

Important domain events:

- `booking.hold_created`
- `booking.hold_expired`
- `booking.confirmed`
- `booking.rescheduled`
- `booking.cancelled`
- `booking.reminder_due`
- `booking.completed`
- `booking.no_show`
- `payment.succeeded`
- `payment.failed`
- `message.inbound_received`
- `message.delivery_failed`

Automation handlers:

| Trigger | Action |
| --- | --- |
| Hold created | Schedule hold expiry job |
| Booking confirmed | Send confirmation, schedule reminders, schedule follow-up |
| Booking rescheduled | Notify customer, cancel old reminder jobs, create new reminder jobs |
| Booking cancelled | Notify customer, release slot, cancel pending jobs |
| Reminder due | Send WhatsApp template or fallback channel |
| Inbound WhatsApp reply | Classify intent, update booking or open staff task |
| Delivery failed | Retry if safe, fall back to SMS/email, alert staff for critical failures |

## WhatsApp integration overview

### Provider options

| Option | Pros | Cons | Best fit |
| --- | --- | --- | --- |
| Meta WhatsApp Cloud API | Direct official integration, fewer middleman costs, full feature access | More setup, more operational responsibility, direct Graph API/token/webhook handling | Product team comfortable owning integration |
| Twilio WhatsApp | Easier onboarding, unified SMS fallback, mature status callbacks | Higher abstraction/cost, provider-specific template tooling | Fast MVP with SMS fallback |
| 360dialog, MessageBird, Vonage, Sinch | Regional support and onboarding help | Varies by region, API differences, vendor lock-in | Businesses needing guided WABA setup |

Recommendation: start with Twilio if the primary goal is fastest reliable MVP
and SMS fallback. Choose Meta Cloud API if controlling cost, owning templates,
and avoiding abstraction are more important.

### WhatsApp constraints that shape the design

- Outbound business-initiated messages generally require approved templates
  when there is no open 24-hour customer service window.
- The 24-hour customer service window opens or resets when the customer sends a
  WhatsApp message to the business.
- Free-form service messages are allowed only during that open window.
- Template status can change after approval; monitor template status webhooks.
- Delivery status is asynchronous: sent, delivered, read, and failed arrive via
  webhooks/status callbacks.
- Webhook handlers must return quickly and process work asynchronously because
  providers retry failed webhook deliveries.
- Meta webhooks should be verified with `X-Hub-Signature-256` using the app
  secret and the raw request body.

### Required WhatsApp templates

Prepare templates early because review can block launch.

Suggested template set:

| Template | Category | Example purpose |
| --- | --- | --- |
| `booking_confirmation` | Utility | Confirm date, time, location, service, and manage-booking link |
| `booking_reminder_24h` | Utility | 24-hour reminder with confirmation/reschedule CTA |
| `booking_reminder_2h` | Utility | Same-day reminder with directions |
| `booking_rescheduled` | Utility | Confirm new date/time |
| `booking_cancelled` | Utility | Confirm cancellation |
| `payment_required` | Utility | Send deposit/payment link |
| `intake_form_request` | Utility | Ask customer to complete intake form |
| `post_booking_followup` | Utility or Marketing | Thank-you/review/rebook prompt; marketing requires explicit opt-in |

Template guidelines:

- Keep copy transactional and specific.
- Include variables only where needed: customer name, service, date/time,
  location, staff, booking reference, and secure manage link.
- Avoid promotional language in utility templates.
- Create language variants for each supported locale.
- Version templates instead of editing high-volume templates right before
  launch.

### Example confirmation template

```text
Hi {{1}}, your {{2}} booking is confirmed for {{3}} at {{4}} with {{5}}.

Location: {{6}}
Manage your booking: {{7}}

Reply HELP if you need assistance.
```

Variables:

1. Customer first name
2. Service name
3. Localized date
4. Localized time
5. Staff or business name
6. Location/address
7. Signed manage-booking URL

## WhatsApp webhook design

Expose a webhook endpoint with two modes:

1. `GET /webhooks/whatsapp`
   - Verify setup challenge.
   - Compare provider verify token.
   - Return challenge on success.

2. `POST /webhooks/whatsapp`
   - Read raw request body.
   - Validate HMAC signature when using Meta Cloud API.
   - Persist webhook payload or normalized event.
   - Return HTTP 200 quickly.
   - Enqueue asynchronous processing.

Normalize provider events into internal events:

| Provider event | Internal event |
| --- | --- |
| Incoming text/button/list reply | `message.inbound_received` |
| Outgoing status sent | `message.sent` |
| Outgoing status delivered | `message.delivered` |
| Outgoing status read | `message.read` |
| Outgoing status failed | `message.delivery_failed` |
| Template approved/rejected/paused | `message.template_status_changed` |

Idempotency:

- Store provider message IDs and webhook event IDs when available.
- If no stable event ID exists, hash provider, event type, message ID, status,
  recipient, and timestamp.
- Make processing safe to replay.
- Never send a new customer-facing message directly inside webhook request
  handling; enqueue a command/job.

## Inbound WhatsApp reply handling

Start with deterministic commands before adding AI.

Supported MVP replies:

| Reply | Action |
| --- | --- |
| `1`, `YES`, `CONFIRM` | Mark customer-confirmed attendance |
| `2`, `RESCHEDULE` | Send manage-booking/reschedule link |
| `3`, `CANCEL` | Send cancellation confirmation flow or link |
| `HELP` | Open staff task and send acknowledgement |
| Unknown | Send short clarification while inside 24-hour window, otherwise notify staff |

Later enhancements:

- Natural-language intent classification.
- Staff handoff inbox.
- FAQ answers for parking, preparation, pricing, and opening hours.
- WhatsApp Flows for structured reschedule/cancel forms if supported by the
  chosen provider.

## Message sending service

Create an internal `MessageService` with a provider adapter interface.

Core operations:

- `sendTemplate(customer, templateName, locale, variables, bookingId)`
- `sendSessionMessage(customer, body, bookingId)`
- `recordInbound(providerPayload)`
- `recordStatus(providerPayload)`
- `getChannelEligibility(customer, messagePurpose)`

Selection rules:

1. If customer has WhatsApp opt-in and phone is valid, use WhatsApp.
2. If within the 24-hour customer service window, free-form replies are allowed.
3. If outside the window, use approved templates only.
4. If WhatsApp fails for critical messages, fall back to SMS/email if available.
5. Suppress non-essential marketing follow-ups unless explicit marketing consent
   exists.

## Security and privacy

- Store phone numbers normalized and encrypted where possible.
- Keep provider access tokens in a secrets manager.
- Validate all webhook signatures.
- Use signed, expiring manage-booking links.
- Do not expose internal booking IDs in public URLs if avoidable.
- Rate limit public booking and webhook endpoints.
- Keep an audit trail for booking changes and message sends.
- Separate transactional WhatsApp consent from marketing consent.
- Provide opt-out handling for marketing messages.

## Observability

Track operational metrics:

- Booking conversion rate from slot view to confirmation.
- Hold expiry rate.
- Double-booking prevention rejections.
- Reminder send success/failure rate.
- WhatsApp template delivery/read/failure rates.
- Payment success/failure rate.
- Webhook processing lag.
- Automation job retry count.
- No-show rate before and after reminders.

Useful alerts:

- WhatsApp delivery failure spike.
- Template status changed to rejected, paused, or disabled.
- Webhook endpoint error rate above threshold.
- Automation job queue backlog.
- Payment webhook failures.
- Availability search latency degradation.

## Implementation milestones

### Milestone 1: Booking foundation

- Define schema/entities.
- Implement service catalog.
- Implement staff/location availability.
- Implement slot search.
- Implement hold creation and expiry.
- Implement booking confirmation.
- Add admin view or API for bookings.

### Milestone 2: Automation engine

- Add domain event log.
- Add background job runner.
- Schedule hold expiry and reminders.
- Add booking event audit trail.
- Add retry and dead-letter handling for failed jobs.

### Milestone 3: WhatsApp outbound

- Choose provider.
- Configure WhatsApp Business Account and phone number.
- Create and submit templates.
- Implement message provider adapter.
- Send booking confirmation and reminders.
- Store provider message IDs and statuses.

### Milestone 4: WhatsApp inbound

- Implement webhook verification.
- Validate signatures.
- Normalize inbound messages/statuses.
- Process deterministic replies.
- Add staff handoff path.
- Add fallback behavior for unknown replies.

### Milestone 5: Production readiness

- Add monitoring and alerts.
- Add admin tools for failed messages/jobs.
- Add privacy/consent controls.
- Run end-to-end tests in provider sandbox/test number.
- Test duplicate webhook deliveries and out-of-order status updates.
- Test timezone, daylight saving, cancellation, reschedule, and payment edge
  cases.

## API surface sketch

Customer-facing:

- `GET /services`
- `GET /availability?serviceId=&staffId=&date=&timezone=`
- `POST /holds`
- `POST /bookings/confirm`
- `GET /bookings/manage/:token`
- `POST /bookings/:id/reschedule`
- `POST /bookings/:id/cancel`

Admin-facing:

- `GET /admin/bookings`
- `PATCH /admin/bookings/:id`
- `GET /admin/messages`
- `POST /admin/messages/:threadId/reply`
- `GET /admin/automation-jobs`
- `POST /admin/automation-jobs/:id/retry`

Webhooks:

- `GET /webhooks/whatsapp`
- `POST /webhooks/whatsapp`
- `POST /webhooks/payments`

## Testing strategy

High-value tests:

- Slot generation respects staff hours, buffers, existing bookings, holds, and
  exceptions.
- Two customers cannot confirm the same slot concurrently.
- Holds expire and release availability.
- Reschedule cancels old reminders and schedules new reminders.
- Cancellation suppresses future reminders.
- WhatsApp template send records provider IDs.
- Duplicate webhook events are idempotent.
- Failed WhatsApp delivery triggers fallback or staff alert.
- Inbound commands update the correct booking.
- Manage-booking links expire and cannot access another customer's booking.

## Key open decisions

1. Which runtime/framework should this greenfield repo use?
2. Which database should be used, and does it support range constraints or
   equivalent locking?
3. Which payment provider is required, if any?
4. Should WhatsApp be direct Meta Cloud API or a provider such as Twilio?
5. Is SMS/email fallback required for critical booking messages?
6. What languages/locales are required for WhatsApp templates?
7. Does the business need multi-location and multi-staff support at launch?
8. Should customers be able to complete the whole booking inside WhatsApp, or
   should WhatsApp link to a web booking flow?

## Recommended next step

For the first implementation pass, scaffold the application with:

- A relational database.
- Booking, hold, customer, service, staff, and message tables.
- A background job mechanism.
- A provider-neutral messaging service.
- WhatsApp Cloud API or Twilio adapter behind that service.

This keeps the booking domain stable even if the WhatsApp provider changes.
