# Functional and System Specification: Infy_EventSphere

## 1. Feature Overview
Infy_EventSphere is an enterprise-grade event management platform designed to orchestrate the entire lifecycle of corporate, educational, and public events. The system spans pre-event planning, room and venue reservation, speaker alignment, participant registration, live event execution with attendance check-in, post-event feedback evaluation, and real-time operational analytics.

The platform is designed around 8 autonomous microservices:
1. **Event Service**: Event authoring, lifecycle management (`DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `PUBLISHED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`), and administrative approval workflows.
2. **Venue Scheduling Service**: Venue/room catalog, capacity thresholds, and collision-free slot reservations.
3. **Speaker Coordination Service**: Speaker profiles, session alignments, and presentation asset management.
4. **Registration Service**: Participant discovery, department-level eligibility verification, capacity counters, and duplicate registration prevention.
5. **Attendance Tracking Service**: Real-time participant check-in, tracking attendance (`ATTENDED` vs `ABSENT`), and no-show analysis.
6. **Feedback Service**: Post-event survey submissions (1–5 rating scale and qualitative reviews), gated by verified attendance.
7. **Notification Service**: Outbox message tracking and event-driven alerts for confirmations, schedule changes, and feedback invitations.
8. **Analytics & Reporting Service**: Real-time materialized metrics on registrations, venue utilization, attendance ratios, and satisfaction scores.

---

## 2. Actors
- **Event Organizer (`ROLE_ORGANIZER`)**: Creates events, schedules sessions, assigns speakers, requests venue slots, monitors registrations, checks in attendees, and reviews event feedback.
- **Participant / Attendee (`ROLE_PARTICIPANT`)**: Discovers events, verifies eligibility, registers online, cancels registrations, downloads session resources, and submits feedback post-attendance.
- **Administrator (`ROLE_ADMIN`)**: Reviews pending event proposals, approves or rejects events, monitors cross-venue utilization, enforces compliance, and views enterprise reporting.
- **Speaker / Presenter (`ROLE_SPEAKER`)**: Views assigned session timings, uploads presentation slides, and accesses session agendas.
- **System / Automated Service**: Emits automated notifications, evaluates schedule collision checks, and tallies analytics snapshots.

---

## 3. Scope

### In-Scope
- End-to-end event lifecycle state machine management.
- Multi-room venue scheduling with conflict detection.
- Departmental eligibility checks (`CORPORATE`, `EDUCATIONAL`) and open access for `PUBLIC` events.
- Atomic participant capacity enforcement and strict duplicate registration prevention.
- Attendance check-in and attendance verification gating for post-event reviews.
- Standardized REST APIs for all 8 microservices with JSON contracts.
- Database-per-service pattern using MySQL 8.x.

### Out-of-Scope (MVP)
- Commercial payment gateway integration (all events are enterprise-sponsored or free in MVP).
- Automated waitlist queue auto-promotion (deferred to post-MVP).
- Native mobile applications (access via responsive web API).
- Biometric hardware integration for check-in (check-in handled via API/web portal).

---

## 4. User Stories

### 4.1 Event Planning & Lifecycle Management
- **US-EV-01**: *As an Event Organizer, I want to create a new event draft specifying title, description, event type (`CORPORATE`, `EDUCATIONAL`, `PUBLIC`), date/time range, and capacity so that I can initiate event planning.*
- **US-EV-02**: *As an Event Organizer, I want to update event details while it is in `DRAFT` status so that I can refine event information before submitting it for review.*
- **US-EV-03**: *As an Event Organizer, I want to submit my drafted event for administrative review so that compliance checks and approvals can take place.*
- **US-EV-04**: *As an Event Organizer, I want to publish an approved event so that it becomes discoverable by eligible participants.*
- **US-EV-05**: *As an Event Organizer, I want to transition the event status to `IN_PROGRESS` when the event starts and `COMPLETED` when it concludes so that execution and feedback workflows are triggered accurately.*
- **US-EV-06**: *As an Event Organizer, I want to cancel an event before execution so that participants are notified and allocated resources are freed.*

### 4.2 Administrative Governance, Compliance & Approvals
- **US-AD-01**: *As an Administrator, I want to view all events awaiting review in a central queue so that I can evaluate them against enterprise policies.*
- **US-AD-02**: *As an Administrator, I want to approve a proposed event so that the organizer can publish it for registration.*
- **US-AD-03**: *As an Administrator, I want to reject a proposed event with specific comments so that the organizer understands the compliance deficiencies.*
- **US-AD-04**: *As an Administrator, I want to override or cancel any event violating enterprise governance so that operational compliance is maintained.*

### 4.3 Venue Allocation & Collision Prevention
- **US-VN-01**: *As an Administrator, I want to manage venue facilities and room inventories with capacity limits so that available physical/virtual spaces are accurately recorded.*
- **US-VN-02**: *As an Event Organizer, I want to search and reserve a room for my event timeframe so that the required physical space is locked in advance.*
- **US-VN-03**: *As an Event Organizer, I want to be immediately notified if a venue is already booked for the chosen time slot so that room conflicts are prevented.*
- **US-VN-04**: *As an Event Organizer, I want the system to ensure my event capacity does not exceed the room's maximum seating capacity so that safety and fire regulations are respected.*

### 4.4 Speaker Coordination & Materials
- **US-SP-01**: *As an Event Organizer, I want to register and manage speaker profiles with subject matter expertise so that presenters can be credited properly.*
- **US-SP-02**: *As an Event Organizer, I want to assign speakers to specific sessions within my event so that schedules reflect presenter commitments.*
- **US-SP-03**: *As a Speaker, I want to upload slide decks, agendas, and handouts so that attendees can access presentation materials.*

### 4.5 Participant Discovery & Online Registration
- **US-RG-01**: *As a Participant, I want to browse and filter published events by category, date, and department so that I can find relevant development and networking opportunities.*
- **US-RG-02**: *As a Participant, I want my department eligibility to be checked automatically during registration so that I am prevented from signing up for sessions intended for other departments.*
- **US-RG-03**: *As a Participant, I want to register for an event with instant confirmation when seats are available so that my seat is reserved immediately.*
- **US-RG-04**: *As a Participant, I want the system to block duplicate registration attempts so that I do not accidentally hold multiple seats for the same event.*
- **US-RG-05**: *As a Participant, I want to receive a clear error when an event is full so that I know no seats remain.*
- **US-RG-06**: *As a Participant, I want to cancel my active registration before the event commences so that my seat is made available to other colleagues.*

### 4.6 Attendance Tracking & Execution
- **US-AT-01**: *As an Event Organizer, I want to view the roster of confirmed registrants when the event begins so that I can conduct participant check-ins.*
- **US-AT-02**: *As an Event Organizer, I want to record attendance status as `ATTENDED` or `ABSENT` for each registrant so that participation is audited.*
- **US-AT-03**: *As an Event Organizer, I want to view live attendance totals during event execution so that I can monitor actual room turnout.*

### 4.7 Post-Event Feedback & Evaluation
- **US-FB-01**: *As an Attended Participant, I want to submit a 1–5 star rating and written comments after the event concludes so that I can share my learning experience.*
- **US-FB-02**: *As a Participant who was absent or unconfirmed, I want the system to prevent me from submitting feedback so that survey integrity is maintained.*
- **US-FB-03**: *As an Attended Participant, I want to be restricted to submitting feedback once per event so that ratings cannot be duplicated or skewed.*
- **US-FB-04**: *As an Event Organizer, I want to view aggregated ratings and qualitative comments for my completed event so that I can evaluate session effectiveness.*

### 4.8 Automated Communications & Notifications
- **US-NT-01**: *As a Participant, I want to receive an automated confirmation email upon successful registration so that I have a written record of my booking.*
- **US-NT-02**: *As a Participant, I want to receive an immediate notification if an event's schedule or venue changes so that I can plan accordingly.*
- **US-NT-03**: *As an Attended Participant, I want to receive a feedback invitation prompt once the event is marked completed so that I am reminded to rate the session.*

### 4.9 Operational Reporting & Real-Time Analytics
- **US-AN-01**: *As an Event Organizer or Batch Owner, I want a real-time summary of total capacity, total registered, slots remaining, and confirmed attendees so that I can evaluate cohort participation.*
- **US-AN-02**: *As an Administrator, I want to view venue utilization dashboards across rooms and dates so that underutilized facilities can be identified.*
- **US-AN-03**: *As an Administrator, I want enterprise-wide operational reports on completion rates, attendance percentages, and participant feedback trends to support strategic planning.*

---

## 5. Functional Requirements (FR)

### Event Service
- **FR-EV-01**: The system shall allow Organizers to create event drafts with title, description, event type (`CORPORATE`, `EDUCATIONAL`, `PUBLIC`), start time, end time, venue ID, and maximum capacity.
- **FR-EV-02**: The system shall enforce event state transitions: `DRAFT` → `PENDING_APPROVAL` → `APPROVED` → `PUBLISHED` → `IN_PROGRESS` → `COMPLETED` / `CANCELLED`.
- **FR-EV-03**: The system shall allow Administrators to approve or reject events with audit remarks.
- **FR-EV-04**: The system shall restrict event modifications exclusively to the event owner (Organizer) while in `DRAFT` status or to Administrators.

### Venue Scheduling Service
- **FR-VN-01**: The system shall manage venue and room inventories, including capacity, location, and amenities.
- **FR-VN-02**: The system shall detect time-slot collisions for room bookings and reject overlapping reservations with HTTP 409 Conflict.
- **FR-VN-03**: The system shall release venue reservations when an associated event is cancelled.

### Speaker Coordination Service
- **FR-SP-01**: The system shall maintain speaker profiles and their subject-matter expertise.
- **FR-SP-02**: The system shall allow Organizers to assign one or more speakers to an event session.
- **FR-SP-03**: The system shall link presentation documents and resource URLs to assigned sessions.

### Registration Service
- **FR-RG-01**: The system shall list all events currently in `PUBLISHED` status for participant discovery.
- **FR-RG-02**: The system shall validate participant department eligibility against the event's target department prior to registration.
- **FR-RG-03**: The system shall atomically check available capacity and decrement available seats upon registration.
- **FR-RG-04**: The system shall strictly reject duplicate registration requests for the same participant and event with HTTP 409 Conflict.
- **FR-RG-05**: The system shall allow participants to cancel their registration while the event is `PUBLISHED`, releasing the allocated capacity.

### Attendance Tracking Service
- **FR-AT-01**: The system shall allow Organizers to record attendance (`ATTENDED` or `ABSENT`) for confirmed registrants once the event reaches `IN_PROGRESS` or `COMPLETED`.
- **FR-AT-02**: The system shall provide attendance status lookup for downstream verification.

### Feedback Service
- **FR-FB-01**: The system shall allow participants to submit ratings (1 to 5) and feedback comments only if the event is `COMPLETED`.
- **FR-FB-02**: The system shall verify via Attendance Service that the participant's status is `ATTENDED` before accepting feedback.
- **FR-FB-03**: The system shall restrict feedback submission to exactly one submission per participant per event.

### Notification Service
- **FR-NT-01**: The system shall log and dispatch notifications upon successful registration, cancellation, and event schedule adjustments.
- **FR-NT-02**: The system shall maintain notification delivery status (`PENDING`, `SENT`, `FAILED`).

### Analytics & Reporting Service
- **FR-AN-01**: The system shall compute real-time registration summaries (total capacity, registered count, seats remaining, attendance count).
- **FR-AN-02**: The system shall compute venue utilization metrics across time windows and facilities for Administrators.
- **FR-AN-03**: The system shall calculate average rating scores and feedback distribution for completed events.

---

## 6. Non-Functional Requirements (NFR)
- **NFR-01 (Performance)**: Read API responses (event discovery, summaries) must respond within 200ms under standard load. Registration write transactions must complete within 500ms.
- **NFR-02 (Data Isolation)**: Strict database-per-service pattern with dedicated MySQL 8.x schemas; zero cross-database queries or shared tables.
- **NFR-03 (Data Integrity & Concurrency)**: Concurrency-safe registration and capacity decrements using optimistic locking (`@Version`) and composite unique constraints on `(event_id, participant_id)`.
- **NFR-04 (Security & Context)**: Role-Based Access Control (RBAC) enforced across endpoints using standard identity headers (`X-User-Id`, `X-User-Role`, `X-User-Department`).
- **NFR-05 (Availability & Fault Tolerance)**: Independent microservice deployment; failure of non-critical services (e.g., Notification or Analytics) must not impair registration or event creation workflows.
- **NFR-06 (Observability & Logging)**: Structured logging with standardized trace headers, request paths, execution timestamps, and error codes.

---

## 7. Validation Rules (VR)
- **VR-01 (Event Title)**: Required, length 3 to 100 characters.
- **VR-02 (Event Description)**: Required, length 10 to 1000 characters.
- **VR-03 (Event Dates)**: `startTime` must be strictly in the future (`> current_timestamp`). `endTime` must be strictly greater than `startTime`.
- **VR-04 (Event Capacity)**: Must be an integer greater than 0, and cannot exceed the assigned room's maximum capacity.
- **VR-05 (Event Type)**: Must be one of `CORPORATE`, `EDUCATIONAL`, `PUBLIC`.
- **VR-06 (Venue Reservation)**: Start and end timestamps must align with room operating hours and cannot overlap with existing bookings.
- **VR-07 (Feedback Rating)**: Must be an integer between 1 and 5 inclusive.
- **VR-08 (Feedback Comments)**: Optional, maximum 500 characters.
- **VR-09 (User Header)**: Requests must contain valid `X-User-Id` and `X-User-Role` headers.

---

## 8. Business Rules (BR)
- **BR-01**: Only an Administrator can transition an event from `PENDING_APPROVAL` to `APPROVED`.
- **BR-02**: An event must be in `APPROVED` state before it can be transitioned to `PUBLISHED`.
- **BR-03**: Only events in `PUBLISHED` status are visible to participants for online registration.
- **BR-04**: A participant from Department "A" cannot register for a `CORPORATE` or `EDUCATIONAL` event targeted strictly to Department "B".
- **BR-05**: Registrations must fail immediately when confirmed registrations equal `max_capacity`.
- **BR-06**: Exactly one active registration record per participant per event is permitted.
- **BR-07**: Attendance check-in can only be performed on confirmed registrations once the event is `IN_PROGRESS` or `COMPLETED`.
- **BR-08**: Feedback submissions are rejected if the participant registration is not marked as `ATTENDED` or if the event has not reached `COMPLETED`.
- **BR-09**: An event cannot be modified once it transitions to `IN_PROGRESS` or `COMPLETED`.

---

## 9. Edge Cases (EC)
- **EC-01 (Concurrent Registrations for Last Seat)**: When two participants attempt to register simultaneously for the final remaining seat, optimistic locking and atomic counters ensure one request succeeds (201 Created) while the other receives `CAPACITY_EXCEEDED` (422/409).
- **EC-02 (Simultaneous Duplicate Requests)**: If a participant double-clicks the register button, the composite unique key `(event_id, participant_id)` in MySQL rejects the second request with `DUPLICATE_REGISTRATION` (409 Conflict).
- **EC-03 (Venue Double-Booking Race Condition)**: If two organizers submit reservations for the same room and time slot, the booking slot unique index rejects the second reservation with `VENUE_SCHEDULE_CONFLICT` (409 Conflict).
- **EC-04 (Cancellation After Event Starts)**: A participant attempting to cancel registration after the event has entered `IN_PROGRESS` is rejected with `INVALID_STATE_TRANSITION` (400 Bad Request).
- **EC-05 (Feedback from Unregistered / Absent User)**: A participant attempting to submit feedback without having attended receives `FEEDBACK_NOT_ELIGIBLE` (422 Unprocessable Entity).
- **EC-06 (Downstream Service Unavailability)**: If `notification-service` is temporarily unreachable during registration, the registration transaction commits successfully, and notification is queued or retried via asynchronous outbox.

---

## 10. Acceptance Criteria (AC)

### AC-01: Event Creation & Approval Workflow
- **Given** an Organizer with `ROLE_ORGANIZER`,
- **When** submitting a valid event draft payload,
- **Then** the event is created with status `DRAFT` and HTTP 201 Created.
- **When** submitted for approval, status changes to `PENDING_APPROVAL`.
- **When** an Admin approves, status becomes `APPROVED`.

### AC-02: Venue Collision Detection
- **Given** an existing confirmed booking for Room R1 between 10:00 AM and 12:00 PM,
- **When** a new booking request is made for Room R1 between 11:30 AM and 1:30 PM,
- **Then** the system rejects the booking with HTTP 409 Conflict and error code `VENUE_SCHEDULE_CONFLICT`.

### AC-03: Participant Department Eligibility
- **Given** a corporate event targeted exclusively to "Engineering",
- **When** a participant with department "Sales" attempts registration,
- **Then** the system rejects the request with HTTP 422 Unprocessable Entity and error code `INELIGIBLE_PARTICIPANT`.

### AC-04: Strict Duplicate Registration Prevention
- **Given** a participant already registered with confirmed status for Event E1,
- **When** the same participant submits a registration request for Event E1,
- **Then** the system rejects the request with HTTP 409 Conflict and error code `DUPLICATE_REGISTRATION`.

### AC-05: Capacity Boundary Enforcement
- **Given** an event with `maxCapacity = 50` and 50 confirmed registrations,
- **When** a new participant attempts registration,
- **Then** the system rejects the request with HTTP 422 Unprocessable Entity and error code `CAPACITY_EXCEEDED`.

### AC-06: Attendance-Gated Feedback Submission
- **Given** an event marked `COMPLETED` and a participant whose attendance record is `ATTENDED`,
- **When** the participant submits a rating of `5` and valid comments,
- **Then** the feedback is recorded with HTTP 201 Created.
- **When** a participant with status `ABSENT` attempts feedback submission,
- **Then** the request is rejected with HTTP 422 Unprocessable Entity and error code `FEEDBACK_NOT_ELIGIBLE`.

---

## 11. Error Codes Catalog

| Error Code | HTTP Status | Description |
|---|---|---|
| `VALIDATION_FAILED` | 400 Bad Request | Payload field validation constraint violated. |
| `INVALID_STATE_TRANSITION` | 400 Bad Request | Requested event state transition is illegal. |
| `UNAUTHORIZED_ACCESS` | 401 Unauthorized | Missing or invalid user identification headers. |
| `FORBIDDEN_OPERATION` | 403 Forbidden | User role lacks permission for the requested action. |
| `RESOURCE_NOT_FOUND` | 404 Not Found | Event, venue, registration, or speaker ID not found. |
| `VENUE_SCHEDULE_CONFLICT` | 409 Conflict | Selected room/venue is already booked during this timeframe. |
| `DUPLICATE_REGISTRATION` | 409 Conflict | Participant is already registered for this event. |
| `INELIGIBLE_PARTICIPANT` | 422 Unprocessable Entity | Participant department does not match event eligibility. |
| `CAPACITY_EXCEEDED` | 422 Unprocessable Entity | Event has reached maximum participant capacity. |
| `FEEDBACK_NOT_ELIGIBLE` | 422 Unprocessable Entity | Participant did not attend the event or event not completed. |
| `INTERNAL_SERVER_ERROR` | 500 Internal Server Error | Unexpected system exception. |

---

## 12. Requirement Traceability Matrix

| Requirement ID | User Story IDs | Acceptance Criteria | Target Microservice | Primary Entity |
|---|---|---|---|---|
| **FR-EV-01, FR-EV-02** | US-EV-01, US-EV-02, US-EV-03, US-EV-04, US-EV-05 | AC-01 | `event-service` | `Event` |
| **FR-EV-03, FR-EV-04** | US-AD-01, US-AD-02, US-AD-03, US-AD-04, US-EV-06 | AC-01 | `event-service` | `EventApproval` |
| **FR-VN-01, FR-VN-02, FR-VN-03** | US-VN-01, US-VN-02, US-VN-03, US-VN-04 | AC-02 | `venue-service` | `Venue`, `Room`, `VenueBookingSlot` |
| **FR-SP-01, FR-SP-02, FR-SP-03** | US-SP-01, US-SP-02, US-SP-03 | N/A | `speaker-service` | `Speaker`, `SpeakerAssignment`, `PresentationResource` |
| **FR-RG-01, FR-RG-03, FR-RG-05** | US-RG-01, US-RG-03, US-RG-05, US-RG-06 | AC-05 | `registration-service` | `Registration` |
| **FR-RG-02** | US-RG-02 | AC-03 | `registration-service` | `ParticipantProfile` |
| **FR-RG-04** | US-RG-04 | AC-04 | `registration-service` | `Registration` |
| **FR-AT-01, FR-AT-02** | US-AT-01, US-AT-02, US-AT-03 | AC-06 | `attendance-service` | `AttendanceRecord` |
| **FR-FB-01, FR-FB-02, FR-FB-03** | US-FB-01, US-FB-02, US-FB-03, US-FB-04 | AC-06 | `feedback-service` | `Feedback` |
| **FR-NT-01, FR-NT-02** | US-NT-01, US-NT-02, US-NT-03 | N/A | `notification-service` | `Notification` |
| **FR-AN-01** | US-AN-01 | N/A | `analytics-service` | `EventAnalyticsSnapshot` |
| **FR-AN-02, FR-AN-03** | US-AN-02, US-AN-03 | N/A | `analytics-service` | `VenueUtilizationMetric` |
