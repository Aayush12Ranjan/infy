# Test Plan and Test Scenarios: Infy_EventSphere

## 1. Purpose of test.md
This document defines the comprehensive test planning, verification scenarios, test strategy, and coverage matrix for the Infy_EventSphere enterprise event management platform. It translates the functional requirements from [spec.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/spec.md), endpoint contracts from [api-contract.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/api-contract.md), and implementation tasks from [tasks.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/tasks.md) into concrete test scenarios covering positive, negative, boundary, and data integrity conditions.

---

## 2. Testing Scope

### 2.1 In-Scope
- **Service Layer Unit Tests**: Pure domain logic, state transition rules, department eligibility checks, capacity calculations, collision detection algorithms, and custom exception throwing.
- **Controller Layer Slice Tests**: HTTP status codes (200, 201, 400, 403, 404, 409, 422), request payload deserialization, response JSON serialization, and header parsing via MockMvc.
- **Validation Tests**: Jakarta Bean Validation constraints on request DTOs (`@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Future`, `@Email`).
- **Data Integrity & Concurrency**: Composite unique constraints (preventing duplicate active registrations and feedback), room booking overlap prevention, and optimistic locking (`@Version`).
- **End-to-End Workflow Verification**: Automated end-to-end event lifecycle flows from draft creation through post-event feedback.

### 2.2 Out-of-Scope Tests
- Payment gateway load and fraud testing (commercial payments are out-of-scope for MVP).
- Physical badge/biometric hardware scanner integration testing.
- Cross-region multi-datacenter disaster recovery stress testing.

---

## 3. Test Strategy
- **Isolation Principle**: Service unit tests run strictly in-memory using **JUnit 5** and **Mockito** with mocked repositories and Feign clients. No live network or external MySQL database is required during unit testing.
- **Controller Slices**: Executed using `@WebMvcTest` focusing exclusively on the web layer, HTTP semantics, and security header evaluations.
- **Deterministic Test Data**: Predefined fixtures ensure tests produce identical, repeatable outcomes regardless of test execution order.
- **Fail-Fast Assertion**: All negative scenarios assert specific domain exception types and structured error codes (e.g., `DUPLICATE_REGISTRATION`, `CAPACITY_EXCEEDED`).

---

## 4. Requirement-to-Test Mapping

| Functional Requirement ID | Target Service | Test Case IDs | Test Type |
|---|---|---|---|
| **FR-EV-01, FR-EV-02** | `event-service` | `TC-EV-01`, `TC-EV-02`, `TC-EV-03` | Service & API |
| **FR-EV-03, FR-EV-04** | `event-service` | `TC-EV-04`, `TC-EV-05`, `TC-EV-06` | Service & API |
| **FR-VN-01, FR-VN-02** | `venue-service` | `TC-VN-01`, `TC-VN-02`, `TC-VN-03` | Service & API |
| **FR-SP-01, FR-SP-02** | `speaker-service` | `TC-SP-01`, `TC-SP-02` | Service & API |
| **FR-RG-01, FR-RG-02** | `registration-service` | `TC-RG-01`, `TC-RG-02` | Service & API |
| **FR-RG-03, FR-RG-04** | `registration-service` | `TC-RG-03`, `TC-RG-04`, `TC-RG-05` | Service & API |
| **FR-RG-05** | `registration-service` | `TC-RG-06` | Service & API |
| **FR-AT-01, FR-AT-02** | `attendance-service` | `TC-AT-01`, `TC-AT-02` | Service & API |
| **FR-FB-01, FR-FB-02** | `feedback-service` | `TC-FB-01`, `TC-FB-02`, `TC-FB-03` | Service & API |
| **FR-NT-01** | `notification-service` | `TC-NT-01` | Service & API |
| **FR-AN-01, FR-AN-02** | `analytics-service` | `TC-AN-01`, `TC-AN-02` | Service & API |

---

## 5. Acceptance Criteria Mapping

| Acceptance Criteria ID | Description | Primary Test Case IDs | Expected Outcome |
|---|---|---|---|
| **AC-01** | Event Creation & Approval Workflow | `TC-EV-01`, `TC-EV-04`, `TC-EV-05` | `DRAFT` created (201), approved by Admin (200), published (200). |
| **AC-02** | Venue Collision Detection | `TC-VN-02`, `TC-VN-03` | Overlap rejected with HTTP 409 `VENUE_SCHEDULE_CONFLICT`. |
| **AC-03** | Participant Department Eligibility | `TC-RG-02` | Mismatched department rejected with HTTP 422 `INELIGIBLE_PARTICIPANT`. |
| **AC-04** | Strict Duplicate Registration Prevention | `TC-RG-04` | Second registration attempt rejected with HTTP 409 `DUPLICATE_REGISTRATION`. |
| **AC-05** | Capacity Boundary Enforcement | `TC-RG-03`, `TC-RG-05` | Registration at max capacity rejected with HTTP 422 `CAPACITY_EXCEEDED`. |
| **AC-06** | Attendance-Gated Feedback Submission | `TC-FB-01`, `TC-FB-02` | Attended user accepted (201); absent user rejected with HTTP 422 `FEEDBACK_NOT_ELIGIBLE`. |

---

## 6. Service Layer Test Scenarios

### 6.1 `event-service` Tests
- **TC-EV-01 (Positive)**: Create event draft with valid parameters; verify status is set to `DRAFT` and timestamps generated.
- **TC-EV-02 (Negative)**: Transition event directly from `DRAFT` to `PUBLISHED` without admin approval; verify `InvalidStateTransitionException` is thrown.
- **TC-EV-03 (Negative)**: Attempt to update event details when event is already in `COMPLETED` status; verify operation rejected.
- **TC-EV-04 (Positive)**: Admin approves pending event with valid remarks; verify status transitions to `APPROVED`.
- **TC-EV-05 (Negative)**: Non-admin user attempts to approve event; verify `ForbiddenException` is thrown.
- **TC-EV-06 (Positive)**: Cancel event in `PUBLISHED` status; verify status becomes `CANCELLED`.

### 6.2 `venue-service` Tests
- **TC-VN-01 (Positive)**: Reserve a room for a free, non-overlapping timeframe; verify booking created with status `CONFIRMED`.
- **TC-VN-02 (Negative - Exact Overlap)**: Request booking for identical room and timeframe as an existing confirmed booking; verify `VenueScheduleConflictException` is thrown.
- **TC-VN-03 (Boundary - Edge Overlap)**: Booking starting exactly when existing booking ends (e.g., Booking 1 ends at 12:00, Booking 2 starts at 12:00); verify booking succeeds without collision.
- **TC-VN-04 (Negative - Partial Overlap)**: Booking starting before and ending during an existing booking; verify `VenueScheduleConflictException` is thrown.

### 6.3 `registration-service` Tests
- **TC-RG-01 (Positive - Eligible Participant)**: Participant from "Engineering" registers for a Corporate event targeted to "Engineering" with open seats; verify status `CONFIRMED`.
- **TC-RG-02 (Negative - Department Mismatch)**: Participant from "Sales" attempts registration for an event restricted to "Engineering"; verify `IneligibleParticipantException` is thrown.
- **TC-RG-03 (Positive - Public Event)**: Participant from any department registers for a `PUBLIC` event; verify registration succeeds.
- **TC-RG-04 (Negative - Duplicate Registration)**: Confirmed participant submits registration for the same event; verify `DuplicateRegistrationException` is thrown.
- **TC-RG-05 (Boundary - Last Seat & Capacity Full)**: Event capacity is 50. 49 participants registered. 50th participant registers successfully. 51st participant registers; verify `CapacityExceededException` is thrown.
- **TC-RG-06 (Positive - Cancellation Frees Seat)**: Participant cancels registration; verify status becomes `CANCELLED`. Subsequent registration succeeds for the vacated seat.

### 6.4 `attendance-service` Tests
- **TC-AT-01 (Positive)**: Organizer marks check-in for confirmed registrant during `IN_PROGRESS` event; verify status becomes `ATTENDED` with check-in timestamp.
- **TC-AT-02 (Negative)**: Check-in recorded for a participant who is not in confirmed registration status; verify `ResourceNotFoundException` is thrown.

### 6.5 `feedback-service` Tests
- **TC-FB-01 (Positive)**: Attended participant submits rating (5) and comment for a `COMPLETED` event; verify feedback stored with HTTP 201.
- **TC-FB-02 (Negative - Did Not Attend)**: Participant marked as `ABSENT` attempts to submit feedback; verify `FeedbackNotEligibleException` is thrown.
- **TC-FB-03 (Negative - Duplicate Feedback)**: Attended participant attempts to submit feedback a second time for the same event; verify `DuplicateFeedbackException` is thrown.
- **TC-FB-04 (Negative - Event Incomplete)**: Attended participant attempts feedback submission while event is still `IN_PROGRESS`; verify `FeedbackNotEligibleException` is thrown.

---

## 7. API / Controller Test Scenarios

### 7.1 Web Layer Tests (`@WebMvcTest`)
- **TC-API-01**: `POST /api/v1/events` with valid body and `ROLE_ORGANIZER` header → verify HTTP `201 Created` and JSON structure.
- **TC-API-02**: `POST /api/v1/events` missing mandatory `title` → verify HTTP `400 Bad Request` with `VALIDATION_FAILED` error code.
- **TC-API-03**: `POST /api/v1/events/{id}/approve` with `ROLE_PARTICIPANT` header → verify HTTP `403 Forbidden`.
- **TC-API-04**: `POST /api/v1/venues/bookings` resulting in schedule collision → verify HTTP `409 Conflict` with `VENUE_SCHEDULE_CONFLICT`.
- **TC-API-05**: `POST /api/v1/registrations` for duplicate attendee → verify HTTP `409 Conflict` with `DUPLICATE_REGISTRATION`.
- **TC-API-06**: `POST /api/v1/registrations` when capacity is full → verify HTTP `422 Unprocessable Entity` with `CAPACITY_EXCEEDED`.
- **TC-API-07**: `POST /api/v1/feedback` with rating `6` → verify HTTP `400 Bad Request` validation error.
- **TC-API-08**: `GET /api/v1/analytics/events/{id}/summary` → verify HTTP `200 OK` with complete summary calculations.

---

## 8. Validation Test Scenarios

| Validation ID | Field Evaluated | Test Input | Expected Behavior |
|---|---|---|---|
| **TC-VAL-01** | `title` | Empty string `""` | Rejected: `@NotBlank` (HTTP 400) |
| **TC-VAL-02** | `title` | String length 2 (`"AI"`) | Rejected: `@Size(min=3, max=100)` (HTTP 400) |
| **TC-VAL-03** | `startTime` | Past timestamp (`2020-01-01T10:00:00Z`) | Rejected: `@Future` (HTTP 400) |
| **TC-VAL-04** | `maxCapacity` | `0` or negative `-5` | Rejected: `@Min(1)` (HTTP 400) |
| **TC-VAL-05** | `rating` | `0` | Rejected: `@Min(1)` (HTTP 400) |
| **TC-VAL-06** | `rating` | `6` | Rejected: `@Max(5)` (HTTP 400) |
| **TC-VAL-07** | `email` | `"invalid-email-string"` | Rejected: `@Email` (HTTP 400) |

---

## 9. Data Integrity & Concurrency Scenarios

- **TC-INT-01 (Composite Unique Constraint - Registration)**: Attempt direct insertion of duplicate active registration records `(eventId=1, participantId='usr_101', status='CONFIRMED')`; verify database `uk_event_participant` integrity constraint violation.
- **TC-INT-02 (Composite Unique Constraint - Feedback)**: Attempt direct insertion of duplicate feedback records `(eventId=1, participantId='usr_101')`; verify database `uk_feedback_event_participant` violation.
- **TC-INT-03 (Optimistic Concurrency - Capacity)**: Two concurrent threads attempt to register for the last remaining seat simultaneously using identical version values; verify exactly one thread commits successfully and the other encounters optimistic lock exception and fails with `CAPACITY_EXCEEDED`.

---

## 10. Manual Verification Scenarios (End-to-End Execution)

### Scenario E2E-01: Full Event & Registration Lifecycle
1. **Organizer** logs in and drafts event `Cloud Architecture Summit 2026` via `POST /api/v1/events`. Status: `DRAFT`.
2. **Organizer** reserves `Auditorium Alpha` via `POST /api/v1/venues/bookings`. Status: `CONFIRMED`.
3. **Organizer** submits event for approval via `POST /api/v1/events/{id}/submit`. Status: `PENDING_APPROVAL`.
4. **Admin** approves event via `POST /api/v1/events/{id}/approve`. Status: `APPROVED`.
5. **Organizer** publishes event via `POST /api/v1/events/{id}/publish`. Status: `PUBLISHED`.
6. **Participant 1** (Eligible) registers online via `POST /api/v1/registrations`. Receives HTTP 201 `CONFIRMED`.
7. **Participant 1** attempts to register again. Receives HTTP 409 `DUPLICATE_REGISTRATION`.
8. **Organizer** marks Participant 1 as `ATTENDED` via `POST /api/v1/attendance/check-in`.
9. **Organizer** marks event status as `COMPLETED`.
10. **Participant 1** submits 5-star rating via `POST /api/v1/feedback`. Receives HTTP 201.
11. **Batch Owner** queries `GET /api/v1/analytics/events/{id}/summary`. Verifies capacity, attendance count, and average rating match exact numbers.

---

## 11. Test Data Requirements

### Users
- `admin_usr`: Role `ROLE_ADMIN`, Department `Corporate Affairs`
- `org_smith`: Role `ROLE_ORGANIZER`, Department `Engineering`
- `usr_dev101`: Role `ROLE_PARTICIPANT`, Department `Engineering`
- `usr_sales201`: Role `ROLE_PARTICIPANT`, Department `Sales`

### Venues & Rooms
- Venue 1: `Global Tech Campus - Block A`, City `Bengaluru`
  - Room 1: `Auditorium Alpha`, Capacity `200`
  - Room 2: `Training Lab 102`, Capacity `30`

---

## 12. Test Execution Order
1. **Bean Validation Tests** (`TC-VAL-*`)
2. **Service Layer Unit Tests** (`TC-EV-*`, `TC-VN-*`, `TC-RG-*`, `TC-AT-*`, `TC-FB-*`)
3. **Controller Web Layer Tests** (`TC-API-*`)
4. **Data Integrity & Concurrency Tests** (`TC-INT-*`)
5. **End-to-End Integration Verification** (`Scenario E2E-01`)

---

## 13. Coverage Matrix

| Test Suite | Total Scenarios | Positive Scenarios | Negative Scenarios | Boundary Scenarios | Mapped Task |
|---|---|---|---|---|---|
| **Event Service** | 6 | 3 | 3 | 0 | `TSK-L01` |
| **Venue Service** | 4 | 1 | 2 | 1 | `TSK-L01` |
| **Registration Service** | 6 | 2 | 3 | 1 | `TSK-L01` |
| **Attendance Service** | 2 | 1 | 1 | 0 | `TSK-L01` |
| **Feedback Service** | 4 | 1 | 3 | 0 | `TSK-L01` |
| **Web Controllers** | 8 | 4 | 4 | 0 | `TSK-M01` |
| **Bean Validations** | 7 | 0 | 7 | 0 | `TSK-J01` |
| **Data Integrity** | 3 | 0 | 3 | 1 | `TSK-C01..05` |
| **E2E Lifecycle** | 1 | 1 | 0 | 0 | `TSK-N01` |
| **Total** | **41** | **13** | **26** | **3** | All |

---

## 14. Open Questions
*No open test ambiguities remain. All state transitions, eligibility boundaries, concurrency locks, and acceptance criteria are fully specified for test execution.*
