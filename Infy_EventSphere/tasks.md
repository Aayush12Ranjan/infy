# Implementation Task Backlog: Infy_EventSphere

## 1. Feature Summary
Infy_EventSphere is an enterprise-wide event management platform providing end-to-end lifecycle orchestration for corporate, educational, and public events across 8 autonomous microservices (`event-service`, `venue-service`, `speaker-service`, `registration-service`, `attendance-service`, `feedback-service`, `notification-service`, `analytics-service`) and an `api-gateway`, backed by MySQL 8.x with a database-per-service pattern.

---

## 2. Implementation Approach
Implementation follows a strict layered, bottom-up dependency order. Each layer is constructed and verified against approved architectural artifacts ([constitution.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/constitution.md), [spec.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/spec.md), [data-model.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/data-model.md), [api-contract.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/api-contract.md)) before proceeding to dependent layers.

---

## 3. Task Dependency Order
The required execution order strictly respects layer dependencies:
```
Project Setup (A)
      │
      ▼
Enums (B)
      │
      ▼
Entities & Database (C)
      │
      ▼
Repositories (D)
      │
      ▼
DTOs (E)
      │
      ▼
Exceptions (F)
      │
      ▼
Mappers (G)
      │
      ▼
Services (H)
      │
      ▼
Controllers (I)
      │
      ▼
Validation (J)
      │
      ▼
Seed Data (K)
      │
      ▼
Unit Tests (L)
      │
      ▼
API Tests (M)
      │
      ▼
Manual / Integration Validation (N)
      │
      ▼
Traceability & Final Review (O)
```

---

## 4. Task List Grouped by Layer

### Section A: Project Setup Tasks

#### TSK-A01: Root Multi-Module Maven Project Setup
- **Title**: Initialize Root Maven Project Structure
- **Description**: Configure parent `pom.xml` with Spring Boot 3.x parent dependencies, Java 17+, dependency management for Spring Cloud, MySQL connector, Lombok/Record support, and Jakarta Validation.
- **Files/Classes to create or update**: `pom.xml`
- **Depends on**: None
- **Related FR IDs**: N/A (Infrastructure)
- **Related AC IDs**: N/A
- **Verification method**: Run `mvn clean validate` from root.
- **Expected output**: Parent POM successfully resolves all common plugins and dependency versions.

#### TSK-A02: Microservice Modules Initialization
- **Title**: Bootstrap Submodules for Microservices and API Gateway
- **Description**: Create module directories and child `pom.xml` files for: `api-gateway`, `event-service`, `venue-service`, `speaker-service`, `registration-service`, `attendance-service`, `feedback-service`, `notification-service`, `analytics-service`.
- **Files/Classes to create or update**: Submodule `pom.xml` files and directory trees.
- **Depends on**: `TSK-A01`
- **Related FR IDs**: N/A
- **Related AC IDs**: N/A
- **Verification method**: Run `mvn compile` across root.
- **Expected output**: All 9 modules compile successfully as Maven submodules.

#### TSK-A03: Application & MySQL Configuration Setup
- **Title**: Configure Microservice Application Profiles & MySQL Database Connections
- **Description**: Configure `application.yml` in each microservice specifying dedicated MySQL database schemas (`eventsphere_event_db`, `eventsphere_venue_db`, etc.), HikariCP settings, server ports (8080–8088), and test profile with in-memory fallback.
- **Files/Classes to create or update**: `application.yml`, `application-test.yml` per service.
- **Depends on**: `TSK-A02`
- **Related FR IDs**: `NFR-02`
- **Related AC IDs**: N/A
- **Verification method**: Spring Boot context startup test per module.
- **Expected output**: Application context resolves database credentials and port allocations correctly.

---

### Section B: Enum Tasks

#### TSK-B01: Core Domain Enumerations Definition
- **Title**: Create Shared and Domain Enums
- **Description**: Define all required domain enumerations: `EventType`, `EventStatus`, `ApprovalDecision`, `BookingStatus`, `RegistrationStatus`, `AttendanceStatus`, `NotificationType`, `NotificationStatus`, and `ResourceType`.
- **Files/Classes to create or update**:
  - `EventType.java`, `EventStatus.java`, `ApprovalDecision.java` in `event-service`
  - `BookingStatus.java` in `venue-service`
  - `ResourceType.java` in `speaker-service`
  - `RegistrationStatus.java` in `registration-service`
  - `AttendanceStatus.java` in `attendance-service`
  - `NotificationType.java`, `NotificationStatus.java` in `notification-service`
- **Depends on**: `TSK-A02`
- **Related FR IDs**: `FR-EV-01`, `FR-EV-02`, `FR-VN-02`, `FR-RG-03`, `FR-AT-01`, `FR-NT-01`
- **Related AC IDs**: `AC-01`, `AC-02`, `AC-06`
- **Verification method**: Maven compile check.
- **Expected output**: All enum types compile with string-backed constants.

---

### Section C: Entity and Database Tasks

#### TSK-C01: Event Service Entities
- **Title**: Implement `Event` and `EventApproval` JPA Entities
- **Description**: Create entities with primary keys, enum string mappings, timestamps, optimistic locking `@Version`, and audit tracking.
- **Files/Classes to create or update**: `Event.java`, `EventApproval.java`
- **Depends on**: `TSK-B01`, `TSK-A03`
- **Related FR IDs**: `FR-EV-01`, `FR-EV-02`, `FR-EV-03`
- **Related AC IDs**: `AC-01`
- **Verification method**: Hibernate schema validation check.
- **Expected output**: MySQL table mappings generate correct DDL with indexes.

#### TSK-C02: Venue Service Entities
- **Title**: Implement `Venue`, `Room`, and `VenueBookingSlot` JPA Entities
- **Description**: Create entities with foreign key mappings, composite unique constraints (`uk_venue_room`), and booking time-window indexes.
- **Files/Classes to create or update**: `Venue.java`, `Room.java`, `VenueBookingSlot.java`
- **Depends on**: `TSK-B01`, `TSK-A03`
- **Related FR IDs**: `FR-VN-01`, `FR-VN-02`
- **Related AC IDs**: `AC-02`
- **Verification method**: Hibernate schema validation.
- **Expected output**: Venue and booking tables created with unique constraints.

#### TSK-C03: Speaker Service Entities
- **Title**: Implement `Speaker`, `SpeakerAssignment`, and `PresentationResource` JPA Entities
- **Description**: Model speaker bios, session schedules, and resource attachments.
- **Files/Classes to create or update**: `Speaker.java`, `SpeakerAssignment.java`, `PresentationResource.java`
- **Depends on**: `TSK-B01`, `TSK-A03`
- **Related FR IDs**: `FR-SP-01`, `FR-SP-02`, `FR-SP-03`
- **Related AC IDs**: N/A
- **Verification method**: Hibernate schema check.
- **Expected output**: Tables created with referential integrity.

#### TSK-C04: Registration Service Entities
- **Title**: Implement `Registration` and `ParticipantProfile` JPA Entities
- **Description**: Implement registration entity with composite unique constraint `uk_event_participant` on `(eventId, participantId, status)` to prevent duplicate active seats.
- **Files/Classes to create or update**: `Registration.java`, `ParticipantProfile.java`
- **Depends on**: `TSK-B01`, `TSK-A03`
- **Related FR IDs**: `FR-RG-02`, `FR-RG-03`, `FR-RG-04`
- **Related AC IDs**: `AC-03`, `AC-04`, `AC-05`
- **Verification method**: Hibernate schema check.
- **Expected output**: Unique constraint enforced on `(event_id, participant_id, status)`.

#### TSK-C05: Attendance, Feedback, Notification, and Analytics Entities
- **Title**: Implement Execution & Evaluation Entities
- **Description**: Implement `AttendanceRecord` (unique event+participant), `Feedback` (unique event+participant, rating 1–5), `Notification` (outbox), and `EventAnalyticsSnapshot`.
- **Files/Classes to create or update**:
  - `AttendanceRecord.java` in `attendance-service`
  - `Feedback.java` in `feedback-service`
  - `Notification.java` in `notification-service`
  - `EventAnalyticsSnapshot.java` in `analytics-service`
- **Depends on**: `TSK-B01`, `TSK-A03`
- **Related FR IDs**: `FR-AT-01`, `FR-FB-01`, `FR-NT-01`, `FR-AN-01`
- **Related AC IDs**: `AC-06`
- **Verification method**: Hibernate schema check.
- **Expected output**: Remaining tables created with appropriate constraints.

---

### Section D: Repository Tasks

#### TSK-D01: Event & Venue Repositories
- **Title**: Implement Repositories for Event & Venue Domains
- **Description**: Extend `JpaRepository`. Add collision detection query in `VenueBookingSlotRepository` checking for overlapping confirmed time intervals.
- **Files/Classes to create or update**: `EventRepository.java`, `EventApprovalRepository.java`, `VenueRepository.java`, `RoomRepository.java`, `VenueBookingSlotRepository.java`
- **Depends on**: `TSK-C01`, `TSK-C02`
- **Related FR IDs**: `FR-EV-01`, `FR-VN-02`
- **Related AC IDs**: `AC-01`, `AC-02`
- **Verification method**: Repository slice test (`@DataJpaTest`).
- **Expected output**: Collision detection query correctly identifies overlapping ranges.

#### TSK-D02: Registration & Attendance Repositories
- **Title**: Implement Repositories for Registration & Attendance Domains
- **Description**: Add queries to count confirmed registrations by event, check duplicate participant registrations, and lookup check-in status.
- **Files/Classes to create or update**: `RegistrationRepository.java`, `ParticipantProfileRepository.java`, `AttendanceRepository.java`
- **Depends on**: `TSK-C04`, `TSK-C05`
- **Related FR IDs**: `FR-RG-03`, `FR-RG-04`, `FR-AT-01`
- **Related AC IDs**: `AC-04`, `AC-05`, `AC-06`
- **Verification method**: Repository slice test (`@DataJpaTest`).
- **Expected output**: Seat count and duplicate existence queries function as expected.

#### TSK-D03: Feedback, Notification & Analytics Repositories
- **Title**: Implement Repositories for Auxiliary Domains
- **Description**: Add repository methods for feedback aggregation, outbox polling, and materialized analytics snapshots.
- **Files/Classes to create or update**: `FeedbackRepository.java`, `NotificationRepository.java`, `AnalyticsSnapshotRepository.java`
- **Depends on**: `TSK-C05`
- **Related FR IDs**: `FR-FB-01`, `FR-NT-01`, `FR-AN-01`
- **Related AC IDs**: `AC-06`
- **Verification method**: Repository slice test (`@DataJpaTest`).
- **Expected output**: Aggregation methods (average rating, total counts) execute properly.

---

### Section E: DTO Tasks

#### TSK-E01: Request & Response DTOs Implementation
- **Title**: Create Complete DTO Schema Catalog
- **Description**: Implement immutable DTOs (using Java records or POJOs) strictly isolating API contracts from entities:
  - Event: `CreateEventRequest`, `UpdateEventRequest`, `ApprovalRequest`, `EventDetailResponse`, `EventSummaryResponse`
  - Venue: `CreateVenueRequest`, `CreateRoomRequest`, `CreateBookingRequest`, `VenueResponse`, `RoomResponse`, `BookingResponse`
  - Speaker: `CreateSpeakerRequest`, `AssignSpeakerRequest`, `SpeakerResponse`
  - Registration: `RegisterEventRequest`, `RegistrationResponse`
  - Attendance: `RecordAttendanceRequest`, `AttendanceResponse`, `AttendanceVerificationResponse`
  - Feedback: `SubmitFeedbackRequest`, `FeedbackResponse`
  - Analytics: `EventSummaryAnalyticsResponse`, `VenueUtilizationResponse`
  - Common: `PageResponse<T>`, `ErrorResponse`, `ValidationErrorDetail`
- **Files/Classes to create or update**: DTO classes across all submodules.
- **Depends on**: `TSK-B01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Jackson serialization/deserialization test.
- **Expected output**: All DTOs serialize and deserialize to conformant JSON structures.

---

### Section F: Exception Handling Tasks

#### TSK-F01: Domain Exception Classes
- **Title**: Create Custom Typed Domain Exceptions
- **Description**: Implement custom unchecked exceptions: `ResourceNotFoundException`, `VenueScheduleConflictException`, `DuplicateRegistrationException`, `IneligibleParticipantException`, `CapacityExceededException`, `FeedbackNotEligibleException`, `InvalidStateTransitionException`, `UnauthorizedException`, `ForbiddenException`.
- **Files/Classes to create or update**: Exception classes per submodule.
- **Depends on**: `TSK-A02`
- **Related FR IDs**: `FR-VN-02`, `FR-RG-02`, `FR-RG-04`, `FR-FB-02`
- **Related AC IDs**: `AC-02`, `AC-03`, `AC-04`, `AC-05`, `AC-06`
- **Verification method**: Maven compile check.
- **Expected output**: Custom exceptions carry typed error codes and messages.

#### TSK-F02: Centralized Global Exception Handler
- **Title**: Implement `@RestControllerAdvice` in Each Microservice
- **Description**: Intercept custom exceptions, validation errors (`MethodArgumentNotValidException`), and unexpected runtime errors to return uniform `ErrorResponse` payloads.
- **Files/Classes to create or update**: `GlobalExceptionHandler.java` in all submodules.
- **Depends on**: `TSK-E01`, `TSK-F01`
- **Related FR IDs**: `NFR-06`
- **Related AC IDs**: All ACs
- **Verification method**: Web slice test triggering exceptions.
- **Expected output**: Error responses conform strictly to the standardized error schema.

---

### Section G: Mapper Tasks

#### TSK-G01: Entity-DTO Mappers Implementation
- **Title**: Implement Transformation Mappers
- **Description**: Implement dedicated mapper components to translate between JPA Entities and DTOs cleanly without manual controller code.
- **Files/Classes to create or update**: `EventMapper.java`, `VenueMapper.java`, `SpeakerMapper.java`, `RegistrationMapper.java`, `AttendanceMapper.java`, `FeedbackMapper.java`
- **Depends on**: `TSK-C01` through `TSK-C05`, `TSK-E01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Unit tests testing bi-directional field mapping.
- **Expected output**: Perfect data translation between entities and DTOs.

---

### Section H: Service Layer Tasks

#### TSK-H01: Event Service Implementation
- **Title**: Implement `EventService` & Lifecycle State Machine
- **Description**: Enforce draft creation, update restrictions, approval workflows, and status transitions (`DRAFT` → `PENDING_APPROVAL` → `APPROVED` → `PUBLISHED` → `IN_PROGRESS` → `COMPLETED` / `CANCELLED`).
- **Files/Classes to create or update**: `EventService.java`, `EventServiceImpl.java`
- **Depends on**: `TSK-D01`, `TSK-G01`, `TSK-F01`
- **Related FR IDs**: `FR-EV-01`, `FR-EV-02`, `FR-EV-03`, `FR-EV-04`
- **Related AC IDs**: `AC-01`
- **Verification method**: Unit test suite with mocked repositories.
- **Expected output**: State transitions strictly enforced; illegal transitions throw exceptions.

#### TSK-H02: Venue Scheduling Service Implementation
- **Title**: Implement `VenueService` & Collision Detection Logic
- **Description**: Manage venue/room records and implement slot booking with real-time overlap collision detection.
- **Files/Classes to create or update**: `VenueService.java`, `VenueServiceImpl.java`
- **Depends on**: `TSK-D01`, `TSK-G01`, `TSK-F01`
- **Related FR IDs**: `FR-VN-01`, `FR-VN-02`, `FR-VN-03`
- **Related AC IDs**: `AC-02`
- **Verification method**: Unit test simulating booking collisions.
- **Expected output**: Overlapping slot bookings rejected with `VenueScheduleConflictException`.

#### TSK-H03: Registration Service Implementation
- **Title**: Implement `RegistrationService` with Eligibility and Capacity Enforcement
- **Description**: Verify participant department eligibility, enforce optimistic concurrency checks against `maxCapacity`, prevent duplicates, and manage self-cancellation.
- **Files/Classes to create or update**: `RegistrationService.java`, `RegistrationServiceImpl.java`, `EventServiceClient.java`
- **Depends on**: `TSK-D02`, `TSK-G01`, `TSK-F01`
- **Related FR IDs**: `FR-RG-01`, `FR-RG-02`, `FR-RG-03`, `FR-RG-04`, `FR-RG-05`
- **Related AC IDs**: `AC-03`, `AC-04`, `AC-05`
- **Verification method**: Unit tests simulating duplicate and capacity exceeded conditions.
- **Expected output**: Concurrency-safe seat allocation and duplicate prevention.

#### TSK-H04: Attendance & Feedback Services Implementation
- **Title**: Implement Attendance Tracking and Gated Feedback Evaluation
- **Description**: Implement check-in tracking and post-event feedback submission with inter-service attendance verification.
- **Files/Classes to create or update**:
  - `AttendanceService.java`, `AttendanceServiceImpl.java`
  - `FeedbackService.java`, `FeedbackServiceImpl.java`, `AttendanceServiceClient.java`
- **Depends on**: `TSK-D02`, `TSK-D03`, `TSK-G01`, `TSK-F01`
- **Related FR IDs**: `FR-AT-01`, `FR-AT-02`, `FR-FB-01`, `FR-FB-02`, `FR-FB-03`
- **Related AC IDs**: `AC-06`
- **Verification method**: Unit tests checking attendance gating on feedback.
- **Expected output**: Feedback accepted only for completed events and attended users.

#### TSK-H05: Speaker, Notification & Analytics Services Implementation
- **Title**: Implement Auxiliary Domain Services
- **Description**: Implement speaker assignments, notification queueing, and analytics summary calculations.
- **Files/Classes to create or update**: `SpeakerServiceImpl.java`, `NotificationServiceImpl.java`, `AnalyticsServiceImpl.java`
- **Depends on**: `TSK-D03`, `TSK-G01`, `TSK-F01`
- **Related FR IDs**: `FR-SP-01`, `FR-NT-01`, `FR-AN-01`, `FR-AN-02`
- **Related AC IDs**: N/A
- **Verification method**: Unit tests verifying calculations and notifications.
- **Expected output**: Accurate summary metrics and outbox records created.

---

### Section I: Controller Layer Tasks

#### TSK-I01: Event & Venue REST Controllers
- **Title**: Implement Controllers for Event and Venue Services
- **Description**: Expose REST endpoints matching [api-contract.md](file:///C:/Users/HP/.gemini/antigravity/scratch/Infy_EventSphere/api-contract.md). Extract user context from `X-User-*` headers.
- **Files/Classes to create or update**: `EventController.java`, `VenueController.java`, `BookingController.java`
- **Depends on**: `TSK-H01`, `TSK-H02`, `TSK-E01`
- **Related FR IDs**: `FR-EV-01` to `FR-EV-04`, `FR-VN-01` to `FR-VN-03`
- **Related AC IDs**: `AC-01`, `AC-02`
- **Verification method**: `@WebMvcTest` endpoint invocations.
- **Expected output**: HTTP 200/201 responses with valid JSON bodies.

#### TSK-I02: Registration, Attendance & Feedback Controllers
- **Title**: Implement Controllers for Participant Execution Services
- **Description**: Expose registration, check-in, and feedback endpoints with `@Valid` annotations.
- **Files/Classes to create or update**: `RegistrationController.java`, `AttendanceController.java`, `FeedbackController.java`
- **Depends on**: `TSK-H03`, `TSK-H04`, `TSK-E01`
- **Related FR IDs**: `FR-RG-01` to `FR-RG-05`, `FR-AT-01`, `FR-FB-01`
- **Related AC IDs**: `AC-03`, `AC-04`, `AC-05`, `AC-06`
- **Verification method**: `@WebMvcTest` tests.
- **Expected output**: REST endpoints return expected contracts and status codes.

#### TSK-I03: Speaker, Notification, Analytics & Gateway Controllers
- **Title**: Implement Remaining Controllers and Gateway Route Configuration
- **Description**: Expose speaker management, notification queue, and reporting endpoints. Configure Gateway routing.
- **Files/Classes to create or update**: `SpeakerController.java`, `NotificationController.java`, `AnalyticsController.java`, `GatewayRoutingConfig.java`
- **Depends on**: `TSK-H05`, `TSK-E01`
- **Related FR IDs**: `FR-SP-01`, `FR-NT-01`, `FR-AN-01`, `FR-AN-02`
- **Related AC IDs**: N/A
- **Verification method**: Gateway routing and controller slice tests.
- **Expected output**: Gateway routes traffic to all downstream microservices properly.

---

### Section J: Validation Tasks

#### TSK-J01: Jakarta Validation Annotations on DTOs
- **Title**: Annotate Request Payloads with Bean Validation Rules
- **Description**: Add `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Future`, `@Email` across all DTO classes according to `VR-01` through `VR-09`.
- **Files/Classes to create or update**: All request DTO classes.
- **Depends on**: `TSK-E01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Validation unit tests passing invalid payloads.
- **Expected output**: Invalid inputs automatically trigger `400 Bad Request` with field details.

---

### Section K: Data Seeding Tasks

#### TSK-K01: Pre-Seeded Sample Data Initialization
- **Title**: Implement `DataInitializer` for Demo Data Seeding
- **Description**: Use `CommandLineRunner` in each microservice to seed realistic demo data in MySQL: 2 venues, 3 rooms, 3 events across states (`PUBLISHED`, `DRAFT`, `COMPLETED`), 2 speakers, registered participants, check-in records, and feedback.
- **Files/Classes to create or update**: `DataInitializer.java` per microservice.
- **Depends on**: `TSK-D01` through `TSK-D03`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Run application and query database tables.
- **Expected output**: MySQL database tables populated with sample demo data on startup.

---

### Section L: Unit Test Tasks

#### TSK-L01: Service Unit Tests with JUnit 5 & Mockito
- **Title**: Implement Service Layer Unit Test Suites
- **Description**: Write isolated unit tests for all services testing happy paths, business validations, and exception throwing.
- **Files/Classes to create or update**:
  - `EventServiceTest.java` (State machine, permissions)
  - `VenueServiceTest.java` (Collision detection)
  - `RegistrationServiceTest.java` (Eligibility, duplicate rejection, capacity full)
  - `AttendanceServiceTest.java` (Check-in, no-shows)
  - `FeedbackServiceTest.java` (Attendance gating, duplicate feedback)
- **Depends on**: `TSK-H01` through `TSK-H05`
- **Related FR IDs**: `FR-EV-01` to `FR-FB-03`
- **Related AC IDs**: `AC-01` to `AC-06`
- **Verification method**: Run `mvn test` across all service modules.
- **Expected output**: 100% of unit tests pass with high assertion coverage.

---

### Section M: API / Controller Test Tasks

#### TSK-M01: Controller MockMvc Slice Tests
- **Title**: Implement Controller Integration Tests with `@WebMvcTest`
- **Description**: Write MockMvc test cases verifying HTTP status codes (200, 201, 400, 403, 409, 422), header parsing, and JSON response bodies.
- **Files/Classes to create or update**:
  - `EventControllerTest.java`
  - `VenueControllerTest.java`
  - `RegistrationControllerTest.java`
  - `FeedbackControllerTest.java`
  - `AnalyticsControllerTest.java`
- **Depends on**: `TSK-I01`, `TSK-I02`, `TSK-I03`, `TSK-F02`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Run `mvn test -Dtest=*ControllerTest`.
- **Expected output**: All web slice tests execute and pass without errors.

---

### Section N: Integration / Manual Verification Tasks

#### TSK-N01: End-to-End Workflow Verification Script
- **Title**: Automated End-to-End Verification Run
- **Description**: Execute automated shell/curl test suite demonstrating the full event lifecycle: Event Creation → Admin Approval → Publishing → Venue Booking → Participant Registration → Duplicate Prevention Test → Attendance Check-in → Feedback Submission → Summary Report Query.
- **Files/Classes to create or update**: `verify_lifecycle.sh` / `verify_lifecycle.ps1`
- **Depends on**: `TSK-I01` through `TSK-K01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: `AC-01` through `AC-06`
- **Verification method**: Execute script against running services.
- **Expected output**: Complete end-to-end flow succeeds with all HTTP assertions verified.

---

### Section O: Traceability and Review Tasks

#### TSK-O01: Traceability Matrix Verification
- **Title**: Compile System Traceability Matrix
- **Description**: Generate `traceability.md` mapping every user story, functional requirement, acceptance criterion, code class, and test case.
- **Files/Classes to create or update**: `traceability.md`
- **Depends on**: `TSK-L01`, `TSK-M01`, `TSK-N01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Automated audit of requirement IDs in test results.
- **Expected output**: 100% requirement coverage verified.

#### TSK-O02: Production Readiness & Architecture Review
- **Title**: Final System Architecture & Quality Review
- **Description**: Generate `review.md` assessing code quality, security posture, database isolation compliance, and Go/No-Go demo decision.
- **Files/Classes to create or update**: `review.md`
- **Depends on**: `TSK-O01`
- **Related FR IDs**: All FRs
- **Related AC IDs**: All ACs
- **Verification method**: Manual and automated code inspection against `constitution.md`.
- **Expected output**: Final review document certifying system production readiness.
