# Clarification & MVP Decisions: Infy_EventSphere

## Purpose
This document resolves ambiguous, missing, or open requirements identified in [raw-requirement.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/raw-requirement.md). For each ambiguity, it articulates the clarification question and establishes the **MVP Decision** (the simplest reasonable temporary decision suitable for rapid delivery, validation, and demo).

---

## 1. Roles & Access Control

### Q1.1: How will user identity, authentication, and role authorization be established for the API?
- **Ambiguity**: `raw-requirement.md` assumes enterprise identity, but MVP needs immediate testability without an external LDAP/OAuth2 server.
- **MVP Decision**: User context will be provided via standard HTTP headers (e.g., `X-User-Id`, `X-User-Role`, `X-User-Department`). The system recognizes three explicit roles:
  1. `ROLE_ADMIN` (approvals, venue allocation review, compliance, platform reporting)
  2. `ROLE_ORGANIZER` (event authoring, schedule management, speaker assignment, attendance tracking)
  3. `ROLE_PARTICIPANT` (browsing published events, self-registration, viewing registrations, submitting feedback)

### Q1.2: Can an organizer modify or cancel events created by another organizer?
- **Ambiguity**: Ownership boundaries among organizers are unspecified.
- **MVP Decision**: Strict resource ownership. An organizer can only modify, schedule, or cancel events they created (`created_by == current_user`). Users with `ROLE_ADMIN` have administrative override permissions across all events.

---

## 2. Business Rules & Lifecycle Workflows

### Q2.1: What are the exact lifecycle states of an event, and who triggers each transition?
- **Ambiguity**: Lifecycle was described broadly without formal state transition boundaries.
- **MVP Decision**: An event follows this state machine:
  1. `DRAFT`: Created by Organizer. Editable.
  2. `PENDING_APPROVAL`: Submitted by Organizer for review.
  3. `APPROVED`: Admin approves event and validates compliance/venue.
  4. `PUBLISHED`: Opened for participant discovery and registration.
  5. `IN_PROGRESS`: Marked when event execution begins.
  6. `COMPLETED`: Marked when event concludes; unlocks feedback submission.
  7. `CANCELLED`: Terminal state; can be triggered by Organizer or Admin before execution.

### Q2.2: How is participant eligibility verified during registration?
- **Ambiguity**: Open question #1 regarding eligibility parameters.
- **MVP Decision**: For MVP, eligibility is determined by `target_department` and `event_type`:
  - `PUBLIC`: Open to all participants regardless of department.
  - `CORPORATE` / `EDUCATIONAL`: Registration is permitted only if the participant's `department` matches the event's `eligible_department` (or if `eligible_department` is set to `"ALL"`).
  - If ineligible, registration fails immediately with a business validation error.

### Q2.3: Does registration require organizer approval or waitlist handling when at maximum capacity?
- **Ambiguity**: Open questions #2 & #3 regarding waitlists and manual approval.
- **MVP Decision**: Instant self-service registration.
  - If current confirmed registrations < `max_capacity`, registration is immediately set to `CONFIRMED`.
  - If current confirmed registrations == `max_capacity`, registration is rejected with `CAPACITY_EXCEEDED` error.
  - Automated waitlists are deferred to post-MVP to keep the workflow simple and deterministic.

### Q2.4: How are duplicate registrations prevented?
- **Ambiguity**: Mechanism for ensuring duplicate registrations do not occur.
- **MVP Decision**: Enforce both at the business layer and database level:
  - Database composite unique constraint on `(event_id, participant_id)`.
  - Service layer pre-check rejecting active registrations with error code `DUPLICATE_REGISTRATION`.

### Q2.5: What are the rules for participant cancellation and attendance check-in?
- **Ambiguity**: Cancellation window and check-in mechanism.
- **MVP Decision**:
  - **Cancellation**: Participants can cancel their registration while the event is in `PUBLISHED` status. Once cancelled, their slot is freed, and status becomes `CANCELLED`.
  - **Attendance Check-in**: Open question #4 resolved by allowing Event Organizers to mark attendance (`ATTENDED` vs. `ABSENT`) for confirmed registrants during or after `IN_PROGRESS` state via a simple check-in endpoint.

### Q2.6: When and how can feedback be submitted?
- **Ambiguity**: Window and restrictions for post-event surveys.
- **MVP Decision**:
  - Feedback is accepted **only** if the event is `COMPLETED` and the participant's registration status is `ATTENDED`.
  - Each attended participant can submit feedback at most once per event.

---

## 3. Validation Rules

### Q3.1: What are the input validation constraints for event creation and scheduling?
- **MVP Decision**:
  - `title`: Mandatory, 3–100 characters.
  - `description`: Mandatory, 10–1000 characters.
  - `eventType`: Mandatory enum (`CORPORATE`, `EDUCATIONAL`, `PUBLIC`).
  - `startTime`: Mandatory, must be in the future (`> now()`).
  - `endTime`: Mandatory, must be strictly after `startTime`.
  - `venueId`: Mandatory, must reference a valid existing venue.
  - `maxCapacity`: Mandatory integer, minimum `1`, maximum cannot exceed the assigned venue's total capacity.

### Q3.2: What are the input validation constraints for feedback submission?
- **MVP Decision**:
  - `rating`: Mandatory integer between `1` and `5` inclusive.
  - `comments`: Optional text, maximum 500 characters.

---

## 4. Data Persistence & Architecture

### Q4.1: Which database engine and schema strategy should be used?
- **Decision (Updated)**: **MySQL 8.x** relational database.
  - Microservices will maintain independent database schemas in MySQL (e.g., `eventsphere_event_db`, `eventsphere_registration_db`, `eventsphere_feedback_db`) ensuring strict database-per-service pattern and loose coupling.
  - Spring Boot JPA/Hibernate with standard MySQL Dialect (`org.hibernate.dialect.MySQLDialect`), connection pooling (HikariCP), and Flyway or JPA auto-DDL (`update` / `validate`).
  - Standard test profiles can use MySQL testcontainers or an embedded/in-memory instance for isolated offline unit/slice tests while production and demo use MySQL.

### Q4.2: What is the microservices decomposition strategy?
- **Decision (Updated - 8 Autonomous Microservices + API Gateway)**:
  Direct capability mapping with strict database-per-service pattern in MySQL:
  1. **Event Service (`event-service`)**:
     - Manages event lifecycle (Draft, Pending Approval, Approved, Published, In Progress, Completed, Cancelled).
     - Dedicated DB: `eventsphere_event_db`.
  2. **Venue Scheduling Service (`venue-service`)**:
     - Manages venue & room inventories, capacity limits, and schedule collision detection.
     - Dedicated DB: `eventsphere_venue_db`.
  3. **Speaker Coordination Service (`speaker-service`)**:
     - Manages speaker profiles, session assignments, and speaker presentation resources.
     - Dedicated DB: `eventsphere_speaker_db`.
  4. **Registration Service (`registration-service`)**:
     - Manages participant discovery, department eligibility verification, capacity counters, and duplicate registration prevention.
     - Dedicated DB: `eventsphere_registration_db`.
  5. **Attendance Tracking Service (`attendance-service`)**:
     - Manages live participant check-in, tracking attendance per session, and no-show metrics.
     - Dedicated DB: `eventsphere_attendance_db`.
  6. **Feedback Service (`feedback-service`)**:
     - Manages post-event survey submissions, ratings (1–5 scale), comments, and participant feedback eligibility.
     - Dedicated DB: `eventsphere_feedback_db`.
  7. **Notification Service (`notification-service`)**:
     - Manages asynchronous email/in-app notifications, registration confirmations, schedule change alerts, and feedback triggers.
     - Dedicated DB: `eventsphere_notification_db`.
  8. **Analytics & Reporting Service (`analytics-service`)**:
     - Aggregates operational reports, registration summaries, attendance rates, and venue utilization metrics for organizers and administrators.
     - Dedicated DB: `eventsphere_analytics_db`.
  9. **API Gateway (`api-gateway`)**:
     - Unified single entry point routing client requests to the respective microservices, enforcing security headers and centralized CORS.
- Inter-service communication uses RESTful HTTP via Spring Cloud OpenFeign / RestClient with resilience patterns (retry/fallback).

### Q4.3: What are the core Domain Entities owned by each microservice?
- **Decision**: To ensure complete entity coverage and strict boundary separation:
  - **`event-service`**:
    - `Event`: Core event entity (id, title, description, eventType, status, startTime, endTime, venueId, maxCapacity, createdBy).
    - `EventApproval`: Audit trail entity for administrator approvals/rejections (id, eventId, adminId, decision, comments, timestamp).
  - **`venue-service`**:
    - `Venue`: Physical or virtual venue location (id, name, address, contactInfo).
    - `Room`: Specific room within venue (id, venueId, roomNumber, capacity, amenities).
    - `VenueBookingSlot`: Time reservation entity (id, venueId, roomId, eventId, startTime, endTime, status).
  - **`speaker-service`**:
    - `Speaker`: Speaker profile (id, name, bio, email, expertiseArea).
    - `SpeakerAssignment`: Speaker-to-event session mapping (id, speakerId, eventId, sessionTitle, assignedTime).
    - `PresentationResource`: Slide decks and attachments (id, speakerId, eventId, fileUrl, resourceType).
  - **`registration-service`**:
    - `Registration`: Participant event booking (id, eventId, participantId, registrationStatus, registeredAt).
    - `ParticipantProfile`: Cached participant eligibility attributes (id, userId, department, email).
  - **`attendance-service`**:
    - `AttendanceRecord`: Check-in log (id, eventId, participantId, checkInTime, attendanceStatus [ATTENDED, ABSENT]).
  - **`feedback-service`**:
    - `Feedback`: Post-event evaluation (id, eventId, participantId, rating [1-5], comments, submittedAt).
  - **`notification-service`**:
    - `Notification`: Outbox and delivery record (id, recipientUserId, recipientEmail, eventId, templateType, status, sentAt).
  - **`analytics-service`**:
    - `EventAnalyticsSnapshot`: Materialized summary views (id, eventId, totalRegistered, totalAttended, averageRating, venueUtilizationRate).

### Q4.4: How will demo sample data be initialized?
- **MVP Decision**: A dedicated `DataInitializer` or database migration/seed script per microservice pre-seeds:
  - 3 predefined venues and rooms in MySQL.
  - Representative events across lifecycle statuses (`PUBLISHED`, `COMPLETED`, `DRAFT`).
  - Sample participant accounts, registrations, and seed feedback.

---

## 5. API Design Assumptions

### Q5.1: What are the API protocol, base path, and serialization standards?
- **MVP Decision**:
  - RESTful architecture over HTTP/1.1.
  - Base path: `/api/v1`.
  - Content-Type: `application/json;charset=UTF-8`.
  - Resource naming: Pluralized nouns (`/api/v1/events`, `/api/v1/registrations`, `/api/v1/venues`).

### Q5.2: How should summary and reporting endpoints be structured?
- **MVP Decision**:
  - `GET /api/v1/events/{id}/registrations/summary`: Returns total capacity, total registered, slots remaining, and attendance count for organizers and batch owners.
  - `GET /api/v1/admin/reports/venue-utilization`: Aggregated venue booking statistics.

---

## 6. Error Handling Strategy

### Q6.1: What is the uniform error response contract?
- **MVP Decision**: All API errors (client and server) return a consistent JSON payload:
  ```json
  {
    "timestamp": "2026-10-28T00:30:00Z",
    "status": 400,
    "error": "Bad Request",
    "errorCode": "DUPLICATE_REGISTRATION",
    "message": "Participant has already registered for this event.",
    "path": "/api/v1/registrations",
    "validationErrors": []
  }
  ```

### Q6.2: What specific HTTP status codes correspond to business exceptions?
- `400 BAD REQUEST`: Input validation failure, malformed payload, invalid state transition.
- `403 FORBIDDEN`: Insufficient role or unauthorized access to another organizer's resource.
- `404 NOT FOUND`: Entity ID not found (Event, Venue, Registration).
- `409 CONFLICT`: Venue schedule overlap or duplicate registration attempt.
- `422 UNPROCESSABLE ENTITY`: Business rule violations (ineligible participant, capacity full, event not completed for feedback).

---

## 7. Test Assumptions

### Q7.1: What test coverage and frameworks are expected?
- **MVP Decision**:
  - Framework: JUnit 5 (`org.junit.jupiter`) and Mockito.
  - Service Layer: Pure unit tests mocking repository dependencies.
  - Web/API Layer: `@WebMvcTest` validating HTTP status codes, request validation, headers, and serialized JSON responses.
  - Test scope covers:
    - Positive happy paths (create, register, check-in, feedback).
    - Business rule validation (venue collision detection, department eligibility, duplicate registration, capacity boundary).
    - Error contract conformity across all negative paths.
