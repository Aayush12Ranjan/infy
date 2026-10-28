# Engineering Constitution: Infy_EventSphere

## Purpose
This document defines the non-negotiable architectural and engineering standards governing all backend microservices for the Infy_EventSphere platform. Every service, module, and pull request must strictly comply with the rules established below.

---

## 1. Architectural Principles & Microservices Boundary Standards

### 1.1 Microservice Independence & Database-per-Service
- Each of the 8 core domain capabilities is an autonomous Spring Boot microservice with its own dedicated MySQL 8.x schema:
  1. `event-service` → `eventsphere_event_db` (Entities: `Event`, `EventApproval`)
  2. `venue-service` → `eventsphere_venue_db` (Entities: `Venue`, `Room`, `VenueBookingSlot`)
  3. `speaker-service` → `eventsphere_speaker_db` (Entities: `Speaker`, `SpeakerAssignment`, `PresentationResource`)
  4. `registration-service` → `eventsphere_registration_db` (Entities: `Registration`, `ParticipantProfile`)
  5. `attendance-service` → `eventsphere_attendance_db` (Entities: `AttendanceRecord`)
  6. `feedback-service` → `eventsphere_feedback_db` (Entities: `Feedback`)
  7. `notification-service` → `eventsphere_notification_db` (Entities: `Notification`)
  8. `analytics-service` → `eventsphere_analytics_db` (Entities: `EventAnalyticsSnapshot`)
  - In addition, an `api-gateway` handles unified request routing, rate limiting, and CORS.
- **Zero Cross-Database Joins**: Direct database access across microservice boundaries is strictly prohibited.
- Communication between microservices must occur via well-defined RESTful APIs (using Spring Cloud OpenFeign or Spring 6 `RestClient`) with appropriate timeout, retry, and fallback policies.

### 1.2 Strict Layered Architecture
Within every microservice, code must follow a clean, four-layer structure with clear boundaries and unidirectional dependencies:
```
[ Controller Layer ]
        │  (Uses DTOs only)
        ▼
[ Service Layer ] (Business Rules, Transactions, Feign Clients)
        │  (Translates DTOs <-> Entities via Mappers)
        ▼
[ Repository Layer ] (Spring Data JPA)
        │  (Operates exclusively on Entities)
        ▼
[ Database Layer ] (MySQL 8.x)
```

---

## 2. Controller Layer Responsibilities

1. **Protocol & Routing Only**: Controllers serve strictly as HTTP entry points (`@RestController`). They parse HTTP requests, map headers, and delegate immediately to the service layer.
2. **Zero Business Logic**: Controllers must never contain business rules, validation calculations, filtering, or database calls.
3. **DTO Exclusivity**:
   - Controllers must never accept or return JPA `@Entity` objects.
   - Controllers only accept `@Valid` request DTOs and return response DTOs wrapped in `ResponseEntity<T>`.
4. **Standardized HTTP Statuses**:
   - `GET`: `200 OK`
   - `POST` (Creation): `201 Created` with `Location` header or resource payload.
   - `PUT` / `PATCH`: `200 OK`
   - `DELETE`: `204 No Content` or `200 OK`
5. **Security & Context Extraction**:
   - Extract user identity and role from incoming headers (`X-User-Id`, `X-User-Role`, `X-User-Department`) and pass them cleanly to service layer parameters.

---

## 3. Service Layer Responsibilities

1. **Business Logic Encapsulation**: All business workflows, domain logic, state machine transitions, eligibility validations, and calculations reside exclusively in `@Service` classes.
2. **Layer Orchestration**: Services orchestrate entity operations via repositories, coordinate inter-service calls via Feign/HTTP clients, and invoke mappers.
3. **Transaction Ownership**: The service layer is the sole owner of transaction boundaries (`@Transactional`).
4. **Exception Propagation**: Services never swallow exceptions. When business constraints are violated, services throw typed custom domain runtime exceptions (e.g., `DuplicateRegistrationException`, `CapacityExceededException`, `VenueScheduleConflictException`).
5. **No Direct HTTP Dependencies**: Service interfaces and implementations must remain decoupled from HTTP servlet APIs (`HttpServletRequest`, `HttpServletResponse`, `ResponseEntity`).

---

## 4. Repository Layer Responsibilities

1. **Spring Data JPA Standards**: Repositories must extend `JpaRepository<Entity, ID>` or `JpaSpecificationExecutor<Entity>`.
2. **Entity Isolation**: Repositories operate solely on JPA entities. They must never accept or return DTOs.
3. **Query Conventions**:
   - Favor Spring Data derived query methods (`findByEventIdAndStatus`).
   - For complex queries, use explicit `@Query` with JPQL. Native SQL queries should only be used when strictly required for MySQL-specific performance optimizations.
   - All write operations modifying multiple records must be annotated with `@Modifying`.
4. **Optimistic Locking**: Critical concurrent entities (such as Event capacity and Venue reservations) must include `@Version` to safeguard against race conditions and lost updates.

---

## 5. Data Transfer Objects (DTO) Standards

1. **Complete Separation**: Request and Response DTOs must be completely isolated from JPA entities.
2. **Immutability & Safety**:
   - DTOs should be immutable (Java `record` classes are preferred for Spring Boot 3 / Java 17+, or standard POJOs with getters and constructors).
   - DTOs must never carry JPA annotations (`@Entity`, `@Table`, `@ManyToOne`, etc.).
3. **Explicit Mapping**:
   - Mapping between DTOs and entities must be executed through dedicated mapper classes (e.g., MapStruct or explicit mapper helper components). Manual mapping in controllers is strictly forbidden.
4. **Granularity**: Separate DTOs must be created for creation (`CreateEventRequest`), updates (`UpdateEventRequest`), and views (`EventSummaryResponse`, `EventDetailResponse`).

---

## 6. Validation Standards

1. **Declarative Bean Validation**:
   - Request DTOs must use Jakarta Validation annotations (`@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Future`, `@Email`).
   - Controller methods must annotate request payloads with `@Valid`.
2. **Fail Fast at the Boundary**:
   - Syntactic and format validation must occur at the controller boundary before reaching the service layer.
3. **Domain Business Validation**:
   - Semantic and stateful validations (e.g., "is venue already booked", "is user eligible for this department-locked event") must be validated in the service layer prior to state mutations.

---

## 7. Exception Handling Standards

1. **Centralized Global Exception Handler**:
   - Every microservice must implement a `@RestControllerAdvice` class intercepting exceptions and converting them to standardized JSON error responses.
2. **Standard Error Contract**:
   Every error response must adhere to the uniform schema:
   ```json
   {
     "timestamp": "ISO-8601 UTC timestamp",
     "status": 409,
     "error": "Conflict",
     "errorCode": "VENUE_SCHEDULE_CONFLICT",
     "message": "The selected venue is already reserved during the requested timeframe.",
     "path": "/api/v1/events",
     "validationErrors": []
   }
   ```
3. **Standard Status Code Mapping**:
   - `MethodArgumentNotValidException` → `400 Bad Request` with field-level `validationErrors`.
   - `ResourceNotFoundException` → `404 Not Found`.
   - `UnauthorizedException` / `ForbiddenException` → `401 Unauthorized` / `403 Forbidden`.
   - `DuplicateResourceException`, `ConflictException` → `409 Conflict`.
   - `BusinessRuleViolationException` → `422 Unprocessable Entity`.
   - Unhandled exceptions → `500 Internal Server Error` (with sanitized logging, never leaking raw stack traces to clients).

---

## 8. Transaction Management Standards

1. **Declarative Transactions**:
   - Apply `@Transactional` at the service class or method level.
   - Read-only service methods must be annotated with `@Transactional(readOnly = true)` to optimize MySQL/Hibernate query execution and dirty-checking overhead.
2. **Transaction Scope**:
   - Transactions must be kept as short as possible. Inter-service REST calls or slow external operations must not be wrapped inside database transactions to prevent MySQL connection starvation.
3. **Rollback Policy**:
   - Rollback must trigger by default on all unchecked (`RuntimeException`) exceptions (`@Transactional(rollbackFor = Exception.class)`).

---

## 9. Testing Standards

1. **Test Framework Stack**:
   - **JUnit 5** (`org.junit.jupiter.api.*`) for all test suites.
   - **Mockito** (`org.mockito.*`) for mocking dependencies.
   - **MockMvc** for controller web-layer testing.
2. **Unit Test Requirements**:
   - Every Service class must have a dedicated unit test suite testing:
     - Positive execution paths.
     - Negative business rule violations (throwing expected custom domain exceptions).
     - Boundary conditions (edge capacity, date overlaps).
3. **Controller Slice Tests**:
   - Controllers must be tested with `@WebMvcTest` verifying:
     - HTTP request serialization and deserialization.
     - HTTP status code adherence.
     - Bean validation failure behavior (`400 Bad Request`).
4. **Test Independence**:
   - Tests must be completely deterministic, repeatable, and run independently of external live environments.
