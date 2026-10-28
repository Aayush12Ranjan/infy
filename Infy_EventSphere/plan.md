# Architecture & Technical Plan: Infy_EventSphere

## 1. Project Overview
Infy_EventSphere is an enterprise-scale distributed event management platform designed to orchestrate corporate, educational, and public events throughout their complete lifecycle. The platform streamlines event drafting, administrative governance and compliance approvals, conflict-free venue reservation, speaker and resource coordination, participant eligibility verification, atomic self-service registration, live attendance tracking, attendance-gated feedback analytics, and real-time operational reporting.

The system is engineered as an autonomous microservices-based ecosystem built on **Java 17+ / Spring Boot 3.x**, backed by **MySQL 8.x** following the database-per-service pattern, and fronted by an **API Gateway**.

---

## 2. Functional Requirement Summary
The architecture fulfills the following core functional domains defined in [spec.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/spec.md):
- **Event Lifecycle Orchestration (`FR-EV`)**: Seven-state state machine managing event drafting, approval requests, administrative authorization, publishing, execution tracking, and cancellation.
- **Venue & Room Scheduling (`FR-VN`)**: Multi-room venue inventory management and collision-free slot booking with conflict detection.
- **Speaker & Presentation Coordination (`FR-SP`)**: Speaker profiling, session scheduling, and presentation asset management.
- **Participant Registration & Eligibility (`FR-RG`)**: Searchable event catalog, department-based eligibility checks, atomic seat decrements, duplicate registration prevention, and self-cancellation.
- **Attendance & Execution (`FR-AT`)**: Live check-in roster, participant status marking (`ATTENDED` vs `ABSENT`), and turnout calculation.
- **Attendance-Gated Feedback (`FR-FB`)**: 1–5 star ratings and qualitative reviews unlocked strictly for verified attendees of completed events.
- **Automated Notifications (`FR-NT`)**: Outbox message tracking and event notification delivery for confirmations, schedule updates, and survey invitations.
- **Operational Analytics & Reporting (`FR-AN`)**: Real-time batch summaries, venue utilization dashboards, and enterprise participation metrics.

---

## 3. Architecture Overview

### 3.1 Distributed Topology
The platform employs a microservices pattern where each domain capability is encapsulated in an independent Spring Boot service with its own dedicated MySQL database schema:

```
                          [ Client Applications ]
                                    │
                                    ▼
                          [ API Gateway : 8080 ]
                                    │ (Reverse Proxy, CORS, Auth Headers)
      ┌─────────────┬──────────────┼─────────────┬─────────────┐
      ▼             ▼              ▼             ▼             ▼
[event-service] [venue-service] [speaker-service] [registration] [attendance]
    :8081          :8082           :8083            :8084           :8085
      │              │               │                │               │
  (MySQL DB)     (MySQL DB)      (MySQL DB)       (MySQL DB)      (MySQL DB)
eventsphere_   eventsphere_    eventsphere_     eventsphere_    eventsphere_
  event_db       venue_db        speaker_db     registration_db  attendance_db
      │              │               │                │               │
      └──────────────┴───────────────┼────────────────┴───────────────┘
                                     ▼
                    ┌────────────────┴────────────────┐
                    ▼                                 ▼
           [feedback-service]             [analytics-service]
                 :8086                          :8087
                   │                              │
              (MySQL DB)                     (MySQL DB)
          eventsphere_feedback_db        eventsphere_analytics_db
                                                  ▲
                                                  │
                                        [notification-service]
                                                :8088
                                                  │
                                             (MySQL DB)
                                       eventsphere_notification_db
```

### 3.2 Inter-Service Communication
- **Synchronous Remote Invocations**: Spring Cloud OpenFeign / Spring 6 `RestClient` for low-latency operational checks (e.g., `registration-service` querying `event-service` for event status and capacity limits; `feedback-service` querying `attendance-service` to verify attendee status).
- **Resilience & Fault Tolerance**: Inter-service client calls utilize request timeouts (connect timeout 2s, read timeout 3s) with graceful degradation to prevent cascade failures.
- **Database Boundary Invariant**: Strict database-per-service model; zero direct cross-service database access.

---

## 4. Layer Design Standards
In accordance with [constitution.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/constitution.md), every microservice adheres to a strict 4-layer architecture:

1. **Controller Layer (`web`)**:
   - Accepts incoming HTTP REST requests.
   - Performs syntactic request validation via Jakarta Bean Validation (`@Valid`).
   - Extracts security context (`X-User-Id`, `X-User-Role`, `X-User-Department`).
   - Strictly accepts and returns DTOs (never JPA entities).
2. **Service Layer (`service`)**:
   - Contains all business logic, eligibility checks, and state transitions.
   - Manages declarative transaction boundaries (`@Transactional`).
   - Translates domain exceptions into structured business error codes.
   - Coordinates inter-service clients and data mappers.
3. **Repository Layer (`repository`)**:
   - Extends Spring Data `JpaRepository`.
   - Encapsulates database queries, derived query methods, and optimistic locking mechanisms (`@Version`).
   - Operates exclusively on JPA entities.
4. **Persistence Layer (`domain/entity`)**:
   - Maps database tables to JPA entities with proper indexes and composite constraints.

---

## 5. Modules & Component Breakdown

### 5.1 `api-gateway`
- **Component**: Spring Cloud Gateway / Spring MVC Reverse Proxy.
- **Responsibility**: Single unified entry point on port 8080, route dispatching, centralized CORS configuration, and security header validation.

### 5.2 `event-service`
- **Components**: `EventController`, `EventService`, `EventRepository`, `EventApprovalRepository`.
- **Responsibility**: Lifecycle state machine, event drafting, administrative approval processing, and status queries.

### 5.3 `venue-service`
- **Components**: `VenueController`, `RoomController`, `BookingController`, `VenueService`, `BookingSlotRepository`.
- **Responsibility**: Venue/room catalog, time-slot collision detection algorithm, and room reservation lifecycle.

### 5.4 `speaker-service`
- **Components**: `SpeakerController`, `SessionAssignmentController`, `SpeakerService`, `SpeakerRepository`.
- **Responsibility**: Presenter profiles, session scheduling, and presentation asset repository.

### 5.5 `registration-service`
- **Components**: `RegistrationController`, `RegistrationService`, `RegistrationRepository`, `EventServiceClient`.
- **Responsibility**: Department eligibility evaluation, optimistic locking capacity checks, atomic seat allocation, duplicate prevention.

### 5.6 `attendance-service`
- **Components**: `AttendanceController`, `AttendanceService`, `AttendanceRepository`, `RegistrationServiceClient`.
- **Responsibility**: Event day check-ins, participant attendance audit records, and no-show reporting.

### 5.7 `feedback-service`
- **Components**: `FeedbackController`, `FeedbackService`, `FeedbackRepository`, `AttendanceServiceClient`.
- **Responsibility**: Survey submission, attendance verification gating, rating validation, and score compilation.

### 5.8 `notification-service`
- **Components**: `NotificationController`, `NotificationService`, `NotificationOutboxRepository`.
- **Responsibility**: Transactional outbox logging, email/in-app alert templates, and async dispatch simulation.

### 5.9 `analytics-service`
- **Components**: `AnalyticsController`, `AnalyticsAggregationService`, `SnapshotRepository`.
- **Responsibility**: Aggregating real-time registration summaries, venue utilization indices, and satisfaction distributions.

---

## 6. Business Workflows

### 6.1 Event Creation & Approval Workflow
```
Organizer                 event-service                  Admin
    │                          │                           │
    ├─ POST /events (Draft) ──>│                           │
    │  [Creates DRAFT]         │                           │
    │                          │                           │
    ├─ POST /events/{id}/submit>│ (State: PENDING_APPROVAL) │
    │                          ├────── Notify Admin ──────>│
    │                          │                           ├─ POST /events/{id}/approve
    │                          │<───── Approval Recorded ──┤
    │                          │ (State: APPROVED)         │
    │                          │                           │
    ├─ POST /events/{id}/publish>│                         │
    │  [State: PUBLISHED]      │                           │
```

### 6.2 Venue Booking & Collision Detection Workflow
1. Organizer submits booking request with `roomId`, `startTime`, and `endTime`.
2. `venue-service` executes collision detection query:
   `SELECT count(b) FROM VenueBookingSlot b WHERE b.roomId = :roomId AND b.status = 'CONFIRMED' AND (b.startTime < :endTime AND b.endTime > :startTime)`.
3. If count > 0, system throws `VenueScheduleConflictException` returning `409 Conflict`.
4. If count == 0, booking slot is persisted with `status = CONFIRMED` and returned (201 Created).

### 6.3 Participant Registration & Duplicate Prevention Workflow
1. Participant initiates registration (`eventId`, `participantId`, `department`).
2. `registration-service` calls `event-service` to verify event is `PUBLISHED` and retrieves `targetDepartment` and `maxCapacity`.
3. **Eligibility Check**: If `targetDepartment != 'ALL'` and does not match participant department, throw `IneligibleParticipantException` (`422 Unprocessable Entity`).
4. **Duplicate Check**: Query `RegistrationRepository` for existing active registration for `(eventId, participantId)`. If found, throw `DuplicateRegistrationException` (`409 Conflict`).
5. **Capacity Check**: Query count of confirmed registrations. If count >= `maxCapacity`, throw `CapacityExceededException` (`422 Unprocessable Entity`).
6. Atomically persist registration record with status `CONFIRMED`.

### 6.4 Attendance Check-in & Feedback Gating Workflow
1. On event day, Organizer records check-in: `POST /api/v1/attendance/check-in` marking `status = ATTENDED`.
2. Event concludes and Organizer transitions event state to `COMPLETED`.
3. Participant submits feedback: `POST /api/v1/feedback`.
4. `feedback-service` calls `attendance-service` to verify that `status == ATTENDED` for that `(eventId, participantId)`.
5. If verified, feedback is saved. If not verified or absent, rejected with `FEEDBACK_NOT_ELIGIBLE` (`422 Unprocessable Entity`).

---

## 7. Security Approach
- **Identity & Authorization Context**: Identity is propagated downstream from the API Gateway or edge via trusted HTTP headers:
  - `X-User-Id`: Unique identifier of the calling user.
  - `X-User-Role`: Calling user's role (`ROLE_ADMIN`, `ROLE_ORGANIZER`, `ROLE_PARTICIPANT`).
  - `X-User-Department`: User's organizational department.
- **Role-Based Access Control (RBAC)**:
  - Services inspect `X-User-Role` at controller/service boundaries.
  - Access to administrative operations (approval, venue modification) is restricted strictly to `ROLE_ADMIN`.
  - Event modifications are restricted to the creating organizer (`ROLE_ORGANIZER`) or `ROLE_ADMIN`.
- **CORS & Edge Protection**: Centralized CORS rules at the API Gateway restricting allowed origins, headers, and HTTP verbs.

---

## 8. Validation Strategy
- **Two-Tier Validation Model**:
  1. **Syntactic Boundary Validation**: Controller payloads are evaluated using Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Future`). Invalid payloads immediately trigger `MethodArgumentNotValidException` mapped to `400 Bad Request` with field-level details.
  2. **Semantic Business Validation**: Service classes enforce contextual rules (room availability, department eligibility, capacity thresholds, event lifecycle status). Violations throw typed domain runtime exceptions mapped to standardized HTTP status codes (`409`, `422`).

---

## 9. Transaction Strategy
- **Declarative Demarcation**: All service layer mutation methods are annotated with `@Transactional(rollbackFor = Exception.class)`.
- **Optimistic Concurrency Control**: Critical concurrent tables (`Event`, `VenueBookingSlot`) feature `@Version` integer columns to protect against race conditions and lost updates without holding persistent database locks.
- **Short Transaction Lifecycles**: External HTTP/Feign calls are executed outside the database transaction boundary to avoid exhausting MySQL connection pool threads.
- **Read-Only Optimization**: Query methods utilize `@Transactional(readOnly = true)` to disable Hibernate dirty-checking and optimize MySQL read operations.

---

## 10. Logging & Observability Strategy
- **Structured Logging**: SLF4J / Logback producing JSON-formatted log statements with timestamps, severity levels, microservice names, and caller IDs.
- **Traceability via Correlation IDs**: The API Gateway injects an `X-Correlation-Id` header into incoming requests; all downstream microservices propagate this ID through `MDC` (Mapped Diagnostic Context) to enable end-to-end trace correlation.
- **Data Protection**: Sensitive internal database credentials, stack traces, and personal details are sanitized and excluded from client error responses.

---

## 11. Test Strategy
- **Service Layer Unit Testing**:
  - JUnit 5 (`@ExtendWith(MockitoExtension.class)`).
  - Complete isolation with Mockito mocks for repositories and Feign clients.
  - Tests covering happy path mutations, business constraint violations, and edge case exceptions.
- **Controller Layer Slice Testing**:
  - `@WebMvcTest` with `MockMvc`.
  - Verification of request validation, HTTP status codes, header parsing, and JSON serialization.
- **Isolated Integration Profile**:
  - Tests execute using Spring's test profile with H2 or mock MySQL dialects to ensure automated test suites run deterministically in any offline environment without external database dependencies.

---

## 12. Risks and Mitigations

| Identified Risk | Impact | Architectural Mitigation Strategy |
|---|---|---|
| **Concurrent Registration Race Condition** | Overbooking past maximum event capacity. | Database-level unique constraint on `(eventId, participantId)`, atomic seat counting, and optimistic locking (`@Version`). |
| **Simultaneous Venue Booking Clash** | Double-booking the same room during overlapping times. | Overlap query validation within serializable/repeatable read isolation and unique index constraints on confirmed booking slots. |
| **Cascading Inter-Service Failures** | Failure of notification or analytics degrades core registration. | Non-blocking execution, short HTTP timeouts (2s/3s), and fallback defaults for auxiliary reporting services. |
| **Distributed Data Inconsistency** | Out-of-sync status between services. | Explicit state transition validation and idempotent update operations across service boundaries. |

---

## 13. Requirement Mapping Matrix

| Architecture Component | Functional Requirements Mapped | Non-Functional Requirements Mapped |
|---|---|---|
| **`event-service`** | `FR-EV-01`, `FR-EV-02`, `FR-EV-03`, `FR-EV-04` | `NFR-02`, `NFR-03`, `NFR-04` |
| **`venue-service`** | `FR-VN-01`, `FR-VN-02`, `FR-VN-03` | `NFR-01`, `NFR-03` |
| **`speaker-service`** | `FR-SP-01`, `FR-SP-02`, `FR-SP-03` | `NFR-01`, `NFR-04` |
| **`registration-service`**| `FR-RG-01`, `FR-RG-02`, `FR-RG-03`, `FR-RG-04`, `FR-RG-05` | `NFR-01`, `NFR-03`, `NFR-04` |
| **`attendance-service`**  | `FR-AT-01`, `FR-AT-02` | `NFR-01`, `NFR-04` |
| **`feedback-service`**    | `FR-FB-01`, `FR-FB-02`, `FR-FB-03` | `NFR-01`, `NFR-04` |
| **`notification-service`**| `FR-NT-01`, `FR-NT-02` | `NFR-05`, `NFR-06` |
| **`analytics-service`**   | `FR-AN-01`, `FR-AN-02`, `FR-AN-03` | `NFR-01`, `NFR-06` |
| **`api-gateway`**         | Routing, Unified Access | `NFR-01`, `NFR-04`, `NFR-06` |
