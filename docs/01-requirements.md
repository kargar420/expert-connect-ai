# ExpertConnect AI — Requirements

## 1. Problem Statement

Students and aspiring software developers often struggle to find experienced mentors who can provide guidance based on their skills, career goals, budget, and availability. Finding the right mentor manually can be difficult and time-consuming.

ExpertConnect AI aims to solve this problem by helping students discover suitable technical mentors through natural-language search, check their availability, and book paid consultation sessions.

## 2. Actors / Users

### 2.1 Student

**Can:**

- Search and filter mentors based on skills, budget, and availability.
- Book available mentoring sessions and make payments.
- View their own bookings and booking status.
- Attend their confirmed mentoring sessions.

**Cannot:**

- Access or modify another student's bookings.
- Modify a mentor's session price.
- Book an unavailable or already reserved slot.

### 2.2 Mentor

**Can:**

- Create and manage their expert profile.
- Set prices for available session slots.
- Modify availability and pricing for unbooked slots.
- Cancel confirmed bookings, triggering the applicable refund process and notifying affected students.
- View only the necessary information of students who have confirmed bookings with them.

**Cannot:**

- Modify a student's personal information.
- Change the agreed price of an existing booking.
- Access bookings belonging to other mentors.

### 2.3 Administrator

**Can:**

- Review mentor applications and verify submitted credentials.
- Approve or reject mentor applications based on available evidence.
- Investigate reported issues and review relevant platform activities.
- Suspend student or mentor accounts when necessary.
- Review audit logs of administrative actions.

**Cannot:**

- Access users' passwords, refresh tokens, or private credentials.
- Directly mark unsuccessful payments as successful.
- Modify financial transaction records to bypass payment verification.

**Future Enhancement:** AI-assisted mentor assessments may help administrators evaluate technical expertise. However, final approval or rejection must involve administrator review.

## 3. Functional Requirements

### FR-01 — User Registration

**Requirement:**

The system shall allow students and prospective mentors to register using valid credentials.

**Expected Behavior:**

- Users can register with an email address and password.
- Duplicate email registrations are rejected.
- Invalid registration details return appropriate validation errors.
- New mentor accounts must complete the approval process before becoming publicly discoverable.

### FR-02 — Mentor Discovery

**Requirement:**

The system shall allow students to search for suitable mentors based on skills, budget, and availability.

**Expected Behavior:**

- Students can filter mentors by technical skills.
- Students can specify a maximum session budget.
- The system returns mentors matching the requested criteria.
- Inactive, suspended, or unapproved mentors are excluded.
- When no mentors match, the system returns an empty result.

### FR-03 — Session Booking

**Requirement:**

The system shall allow authenticated students to temporarily reserve an available mentoring slot before completing payment while preventing conflicting reservations.

**Expected Behavior:**

- The system verifies that the selected slot is available.
- Only one student can successfully reserve a slot at a time.
- The Marketplace Service shall use PostgreSQL pessimistic row-level locking within a short database transaction to coordinate competing reservation requests.
- The system shall recheck the slot's current state while holding the database lock.
- If available, the system creates a temporary booking hold and records its expiration time.
- If the slot is already reserved or confirmed, the system rejects the booking request.
- Concurrent booking requests must not result in double booking.
- If payment is not completed within the permitted reservation period, the temporary hold expires and the slot becomes available again, subject to payment reconciliation rules.
- A booking becomes confirmed only after verified successful payment and required booking checks.
- The agreed session price is preserved for the booking.
- Late payment confirmations must be reconciled without creating conflicting bookings.
- Repeated submissions of the same logical booking request must not create duplicate booking holds.

**Concepts:** Pessimistic locking, PostgreSQL transactions, concurrency control, idempotency, booking expiration, and database constraints.

## 4. Non-Functional Requirements

### NFR-01 — Performance

**Measurable performance targets will be defined and validated through benchmarking.**

### NFR-02 — Reliability

**Requirement:**

The system shall preserve confirmed bookings even when the Notification Service is unavailable. Pending notifications must be retained and processed after recovery.

**Expected Behavior:**

- Confirmed bookings must remain unaffected by notification failures.
- Pending notification jobs must be stored reliably for later processing.
- After service recovery, the system shall resume processing pending notifications.
- Pending notifications should generally be processed in their original order, subject to retry and priority policies.
- Failed notifications should be retried without blocking other notifications.
- Duplicate event processing must not create duplicate logical notification records.
- Repeatedly failed notifications must be recorded for investigation.
- The original booking confirmation time must be preserved, even if notifications are delivered later.

**Concepts:**

- Asynchronous processing
- Message queues
- FIFO scheduling
- Retry mechanisms
- Idempotency
- Failure recovery

**Important Note:** The system aims for idempotent notification processing. Exactly-once delivery through external email providers cannot always be guaranteed.

### NFR-03 — Security

**Requirement:**

The system shall protect user credentials and sensitive information from unauthorized access through secure authentication, authorization, encrypted communication, and restricted database access.

**Expected Behavior:**

- Passwords must be stored using secure, salted password hashing.
- Sensitive information must be transmitted over encrypted connections.
- Users must only access resources they are authorized to view or modify.
- Database access must be restricted to authorized services.
- Refresh tokens, passwords, and private credentials must not appear in application logs.
- Payment verification must be performed by trusted backend components.

**Future Enhancement:**

- Detect unfamiliar device logins.
- Send security alerts for suspicious authentication activity.
- Allow users to review and revoke active sessions.

**Concepts:**

- Authentication and authorization
- Password hashing
- HTTPS/TLS
- RBAC
- Least privilege
- Secure session management

### NFR-04 — Concurrency and Contention Handling

**Requirement:**

The system shall handle competing booking requests safely, including periods of high contention for popular mentoring slots.

**Expected Behavior:**

- At most one active booking hold or confirmed booking may exist for a particular slot.
- Competing reservation transactions must coordinate using short-lived PostgreSQL row-level locks.
- A transaction must not retain its database lock while waiting for a student to complete payment.
- Contending requests must use a bounded wait or fail-fast locking policy to avoid indefinite waiting.
- Failed booking transactions must release their database locks through commit or rollback.
- The system must avoid unnecessary automatic retries for slots that are already reserved.
- Concurrent booking tests must verify that multiple simultaneous requests cannot create duplicate reservations.
- Lock contention, response times, and transaction failures should be measured during performance testing.

**Initial Testing Scenario:**

Simulate 100 students attempting to reserve the same available slot concurrently. Verify that exactly one active reservation is persisted and no duplicate booking is created.

**Design Assumption:**

Popular mentoring slots may experience high contention. Pessimistic locking is the initial implementation choice, but its performance will be evaluated against alternatives using benchmarks.

**Concepts:**

- Pessimistic locking
- Row-level locks
- Transaction isolation
- Lock contention
- Deadlocks and timeouts
- Concurrency testing

## 5. Main User Journey

### 5.1 Mentor Discovery

1. A student searches for a mentor using filters such as technical skills, experience level, budget, and preferred date/time.
2. The system retrieves matching approved mentors and displays their profiles, session prices, and available slots.
3. The student selects a mentor and an available time slot.

### 5.2 Session Booking

4. The student requests a booking using an idempotency key.
5. The Marketplace Service begins a database transaction and attempts to acquire a row-level lock on the selected slot.
6. After obtaining the lock, the system checks the slot's current availability.
7. If available, the system creates a temporary booking hold, records its expiration time, and commits the transaction.
8. Once the transaction commits, the database lock is released. If the slot is unavailable or the lock cannot be acquired under the configured policy, the request is rejected appropriately.
9. The student proceeds to payment and may include an optional message describing the guidance they need.

### 5.3 Payment and Confirmation

10. The student completes payment through the payment gateway.
11. The backend verifies payment and finalizes the booking if the reservation remains valid.
12. Once confirmed, the student and mentor receive booking notifications containing the scheduled session details.

### 5.4 Mentor Interaction

13. The mentor views the confirmed booking and may send an optional message or instructions to the student.
14. If the mentor cancels the confirmed session, the system initiates the applicable refund process and notifies the student.
15. The student can track the booking, cancellation, and refund status.

### 5.5 Alternative Scenarios

- **Slot unavailable:** The system rejects the booking request.
- **Concurrent reservation:** Only one request successfully creates the reservation.
- **Repeated booking request:** An idempotent retry does not create another booking.
- **Lock contention:** The request follows the configured bounded-wait or fail-fast policy.
- **Transaction failure:** Changes are rolled back and database locks are released.
- **Payment not completed:** The temporary hold expires after the configured reservation period.
- **Payment succeeds after hold expiration:** The system reconciles the payment and booking state; it must not double-book the slot.
- **Notification Service unavailable:** The confirmed booking remains intact, and pending notifications are retried.
- **Mentor cancellation:** The booking is cancelled according to the cancellation policy, and the refund process is tracked until completion.

## 6. Business Rules

### BR-01 — Idempotent Booking Requests

- Repeated submissions of the same logical booking request must not create multiple booking holds.
- Each booking request shall have an idempotency key scoped to the requesting student.
- Reusing the same idempotency key with different request parameters must be rejected.
- Retrying a request should return its existing result or processing status without creating another reservation.

### BR-02 — Prevent Double Booking

- Each slot can have at most one active reservation or confirmed booking.
- The Marketplace Service shall initially use pessimistic row-level locking to coordinate conflicting reservation requests.
- The slot's availability must be checked after acquiring the lock.
- Slot reservation and booking record creation must be committed within the same database transaction.
- Database constraints must provide additional protection against duplicate active bookings.
- Unsuccessful competing requests must not create additional reservations.

**Architectural Decision:**

Pessimistic locking is the initial choice because popular time slots may attract many competing booking attempts. This assumption must be validated through concurrency and performance testing.

### BR-03 — Booking Hold Expiration

- Temporary booking holds shall expire after a configurable period.
- Database locks must not remain active during this reservation period.
- An expired hold must not authorize a new booking confirmation without validation.
- Expired holds shall be released through a safe expiration and reconciliation process.
- Payment confirmations arriving around or after expiration must be handled without double booking.

### BR-04 — Booking Price Integrity

- The agreed session price must be saved when the booking is created.
- Mentors cannot alter the price of an existing reservation or confirmed booking.
- Price changes may apply only to eligible future bookings.

### BR-05 — Payment and Refund Integrity

- A booking must not be confirmed without trusted payment verification.
- Duplicate payment notifications must not create duplicate financial operations.
- Mentor-initiated cancellations must follow the applicable refund policy.
- Refund requests and outcomes must be tracked until resolved.
- Payment failure or reconciliation issues must not result in conflicting confirmed bookings.

## 7. MVP Scope

### 7.1 MVP Goal

In the first version of ExpertConnect AI, students and mentors can register, log in, and manage their profiles. Students can find approved technical mentors using basic skill, budget, and availability filters, reserve an available slot, complete a sandbox payment, and view a confirmed booking. Both participants receive basic booking notifications.

The MVP must also handle repeated booking requests, competing reservations, payment verification, and temporary notification failures safely. **Natural-language/AI search is a later milestone**, not a dependency for the first working booking journey.

### 7.2 Must-Have Features

1. **Authentication and roles:** Student and mentor registration/login, authenticated sessions, and role-based access to protected operations.
2. **Mentor profiles and approval:** Mentors can manage their profiles, skills, session prices, and availability; an administrator can manually approve or reject applications.
3. **Basic mentor discovery:** Students can filter approved, active mentors by skills, maximum price, and available time slots.
4. **Safe slot reservation:** Students can create a temporary booking hold; it expires if payment is not completed. The Marketplace Service prevents double booking using short PostgreSQL transactions and pessimistic row-level locking, supplemented by database constraints.
5. **Sandbox payments:** Create a payment order, integrate a test-mode payment provider, verify payment on the backend, handle duplicate callbacks, and track payment state.
6. **Booking confirmation and history:** Confirm only after verified payment and a valid reservation; students and mentors can view their own booking details and status.
7. **Basic notifications:** Send booking confirmations to both parties. Failed deliveries are retained and retried without cancelling the booking.
8. **Cancellation and refunds:** Support mentor-initiated cancellation with a basic test-mode refund workflow and visible refund status.

**Implementation scope:** A minimal React interface is sufficient. Backend services will be introduced incrementally; the MVP aims for clear ownership boundaries rather than unnecessary infrastructure complexity.

### 7.3 Out of Scope (Future Enhancements)

- AI-powered natural-language queries, embeddings, semantic ranking, and RAG.
- AI-based mentor interviews or automatic mentor scoring.
- Built-in chat, video calls, and sophisticated rescheduling.
- Real-money payouts, mentor commissions, and production financial settlement.
- Complex recommendation personalization, analytics dashboards, and mobile applications.
- Advanced operational tooling beyond the logging, tests, and failure handling needed for a safe MVP.

### 7.4 MVP Success Criteria

- A student can complete an end-to-end journey: **register → search → hold slot → pay in test mode → see confirmed booking**.
- Only approved and active mentors appear in public discovery results.
- When multiple students attempt to reserve one slot simultaneously, **at most one active hold or confirmed booking** exists; test this with 100 concurrent attempts.
- Retrying the same booking request with the same idempotency key does not create a second logical booking.
- Failed or unverified payments do not confirm bookings; late payment results are reconciled without double booking.
- When notifications are temporarily unavailable, the booking remains confirmed and pending notifications can be retried after recovery.
- Mentor cancellation initiates the defined test-mode refund workflow, and the student can see the cancellation/refund status.
- The core booking, payment, and notification failure cases are covered by automated tests; measurable latency targets will be set after initial benchmarking.