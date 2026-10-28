# Data Model Specification: Infy_EventSphere

## Purpose
This document defines the relational data models, entity attributes, keys, constraints, validations, enums, sample seed data, requirement mappings, and JPA mapping guidelines for the microservices comprising the Infy_EventSphere platform. Each microservice manages its own isolated MySQL 8.x schema in adherence to the database-per-service architectural invariant.

---

## 1. Enumerations

### 1.1 `EventType`
- `CORPORATE`: Internal enterprise training, team workshops, and corporate all-hands.
- `EDUCATIONAL`: Academic seminars, technical bootcamps, and skill certification tracks.
- `PUBLIC`: Open-invite community tech meetups and industry conferences.

### 1.2 `EventStatus`
- `DRAFT`: Initial draft created by Organizer; editable.
- `PENDING_APPROVAL`: Submitted by Organizer; awaiting administrative review.
- `APPROVED`: Authorized by Administrator; eligible for publishing.
- `PUBLISHED`: Visible in catalog; open for eligible participant registration.
- `IN_PROGRESS`: Event actively running; check-ins enabled.
- `COMPLETED`: Event concluded; unlocks attendee feedback.
- `CANCELLED`: Terminal state; registrations and venue reservations invalidated.

### 1.3 `ApprovalDecision`
- `APPROVED`: Proposal conforms to enterprise standards and policy.
- `REJECTED`: Proposal fails compliance or resource availability requirements.

### 1.4 `BookingStatus`
- `CONFIRMED`: Active room reservation.
- `CANCELLED`: Released reservation.

### 1.5 `RegistrationStatus`
- `CONFIRMED`: Seat successfully allocated and reserved.
- `CANCELLED`: Registration withdrawn by participant; seat capacity released.

### 1.6 `AttendanceStatus`
- `ATTENDED`: Participant physically or virtually checked in during the event.
- `ABSENT`: Confirmed registrant failed to check in (no-show).

### 1.7 `NotificationType`
- `REGISTRATION_CONFIRMATION`: Triggered upon successful seat booking.
- `REGISTRATION_CANCELLATION`: Triggered upon participant withdrawal.
- `EVENT_UPDATE`: Triggered upon schedule or venue modifications.
- `FEEDBACK_INVITATION`: Triggered upon event completion for attended participants.

### 1.8 `NotificationStatus`
- `PENDING`: Enqueued for dispatch.
- `SENT`: Successfully transmitted to recipient.
- `FAILED`: Delivery failed after maximum retries.

### 1.9 `ResourceType`
- `SLIDE_DECK`: Presentation slides (PDF/PPTX).
- `SYLLABUS`: Session outline and prerequisite documentation.
- `CODE_REPOSITORY`: Link to lab or code assets.

---

## 2. Microservice Entity Definitions

### 2.1 `event-service` (Schema: `eventsphere_event_db`)

#### Entity: `Event`
- **Description**: Central entity tracking event configuration, schedule, capacity, and lifecycle state.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `title`: `VARCHAR(100)`, Not Null, Validated: 3–100 chars.
  - `description`: `VARCHAR(1000)`, Not Null, Validated: 10–1000 chars.
  - `eventType`: `VARCHAR(30)` (Enum: `EventType`), Not Null.
  - `status`: `VARCHAR(30)` (Enum: `EventStatus`), Not Null, Default: `'DRAFT'`.
  - `startTime`: `DATETIME`, Not Null, Must be future date.
  - `endTime`: `DATETIME`, Not Null, Must be strictly after `startTime`.
  - `venueId`: `BIGINT`, Not Null (References external Venue entity).
  - `roomId`: `BIGINT`, Not Null (References external Room entity).
  - `eligibleDepartment`: `VARCHAR(50)`, Not Null, Default: `'ALL'`.
  - `maxCapacity`: `INT`, Not Null, Validated: Min 1, Max <= Room capacity.
  - `createdBy`: `VARCHAR(50)`, Not Null (Organizer User ID).
  - `version`: `INT`, Not Null, Default: 0 (Optimistic Locking).
  - `createdAt`: `DATETIME`, Not Null, System timestamp.
  - `updatedAt`: `DATETIME`, Not Null, System timestamp.
- **Constraints & Indexes**:
  - Index: `idx_event_status` on (`status`).
  - Index: `idx_event_created_by` on (`createdBy`).

#### Entity: `EventApproval`
- **Description**: Audit log tracking administrative review decisions and feedback.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `eventId`: `BIGINT`, Not Null, Foreign Key to `Event(id)`.
  - `adminId`: `VARCHAR(50)`, Not Null (Administrator User ID).
  - `decision`: `VARCHAR(20)` (Enum: `ApprovalDecision`), Not Null.
  - `remarks`: `VARCHAR(500)`, Nullable.
  - `reviewedAt`: `DATETIME`, Not Null, System timestamp.
- **Relationships**:
  - Many-to-One with `Event` (`@ManyToOne`).

---

### 2.2 `venue-service` (Schema: `eventsphere_venue_db`)

#### Entity: `Venue`
- **Description**: Physical building or campus facility.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `name`: `VARCHAR(100)`, Not Null, Unique, Validated: 2–100 chars.
  - `address`: `VARCHAR(255)`, Not Null.
  - `city`: `VARCHAR(100)`, Not Null.
  - `createdAt`: `DATETIME`, Not Null.

#### Entity: `Room`
- **Description**: Individual room, auditorium, or lab within a venue.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `venueId`: `BIGINT`, Not Null, Foreign Key to `Venue(id)`.
  - `roomNumber`: `VARCHAR(50)`, Not Null.
  - `maxSeatingCapacity`: `INT`, Not Null, Validated: Min 1.
  - `amenities`: `VARCHAR(255)`, Nullable (e.g., "Projector, AV System, Whiteboard").
- **Constraints**:
  - Unique Constraint: `uk_venue_room` on (`venueId`, `roomNumber`).
- **Relationships**:
  - Many-to-One with `Venue` (`@ManyToOne`).

#### Entity: `VenueBookingSlot`
- **Description**: Reserved time-window for a room allocated to an event.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `venueId`: `BIGINT`, Not Null.
  - `roomId`: `BIGINT`, Not Null, Foreign Key to `Room(id)`.
  - `eventId`: `BIGINT`, Not Null (References external Event).
  - `startTime`: `DATETIME`, Not Null.
  - `endTime`: `DATETIME`, Not Null.
  - `bookingStatus`: `VARCHAR(20)` (Enum: `BookingStatus`), Not Null, Default: `'CONFIRMED'`.
  - `version`: `INT`, Not Null, Default: 0 (Optimistic Locking).
  - `bookedAt`: `DATETIME`, Not Null.
- **Constraints & Indexes**:
  - Index: `idx_booking_window` on (`roomId`, `bookingStatus`, `startTime`, `endTime`).

---

### 2.3 `speaker-service` (Schema: `eventsphere_speaker_db`)

#### Entity: `Speaker`
- **Description**: Profile details of session presenters and industry speakers.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `fullName`: `VARCHAR(100)`, Not Null.
  - `email`: `VARCHAR(120)`, Not Null, Unique, Validated: Valid Email.
  - `biography`: `VARCHAR(1000)`, Nullable.
  - `expertiseArea`: `VARCHAR(100)`, Not Null.

#### Entity: `SpeakerAssignment`
- **Description**: Mapping of speaker to a specific event session and schedule slot.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `speakerId`: `BIGINT`, Not Null, Foreign Key to `Speaker(id)`.
  - `eventId`: `BIGINT`, Not Null (References external Event).
  - `sessionTitle`: `VARCHAR(150)`, Not Null.
  - `sessionStartTime`: `DATETIME`, Not Null.
  - `sessionEndTime`: `DATETIME`, Not Null.
- **Relationships**:
  - Many-to-One with `Speaker` (`@ManyToOne`).

#### Entity: `PresentationResource`
- **Description**: Slide decks and external reference links provided by the speaker.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `assignmentId`: `BIGINT`, Not Null, Foreign Key to `SpeakerAssignment(id)`.
  - `resourceTitle`: `VARCHAR(100)`, Not Null.
  - `resourceType`: `VARCHAR(30)` (Enum: `ResourceType`), Not Null.
  - `resourceUri`: `VARCHAR(500)`, Not Null.
  - `uploadedAt`: `DATETIME`, Not Null.

---

### 2.4 `registration-service` (Schema: `eventsphere_registration_db`)

#### Entity: `Registration`
- **Description**: Participant event seat booking record.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `eventId`: `BIGINT`, Not Null (References external Event).
  - `participantId`: `VARCHAR(50)`, Not Null (Participant User ID).
  - `participantEmail`: `VARCHAR(120)`, Not Null.
  - `participantDepartment`: `VARCHAR(50)`, Not Null.
  - `status`: `VARCHAR(20)` (Enum: `RegistrationStatus`), Not Null, Default: `'CONFIRMED'`.
  - `registeredAt`: `DATETIME`, Not Null.
  - `cancelledAt`: `DATETIME`, Nullable.
- **Constraints & Indexes**:
  - Unique Constraint: `uk_event_participant` on (`eventId`, `participantId`, `status`).
  - Index: `idx_reg_event_status` on (`eventId`, `status`).

#### Entity: `ParticipantProfile`
- **Description**: Cached local profile for rapid departmental eligibility verification.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `userId`: `VARCHAR(50)`, Not Null, Unique.
  - `fullName`: `VARCHAR(100)`, Not Null.
  - `email`: `VARCHAR(120)`, Not Null.
  - `department`: `VARCHAR(50)`, Not Null.

---

### 2.5 `attendance-service` (Schema: `eventsphere_attendance_db`)

#### Entity: `AttendanceRecord`
- **Description**: Live check-in log recording actual event turnout and participation.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `eventId`: `BIGINT`, Not Null.
  - `participantId`: `VARCHAR(50)`, Not Null.
  - `checkInTime`: `DATETIME`, Nullable.
  - `status`: `VARCHAR(20)` (Enum: `AttendanceStatus`), Not Null, Default: `'ABSENT'`.
  - `verifiedBy`: `VARCHAR(50)`, Not Null (Organizer User ID).
  - `recordedAt`: `DATETIME`, Not Null.
- **Constraints & Indexes**:
  - Unique Constraint: `uk_attendance_event_participant` on (`eventId`, `participantId`).
  - Index: `idx_attendance_status` on (`eventId`, `status`).

---

### 2.6 `feedback-service` (Schema: `eventsphere_feedback_db`)

#### Entity: `Feedback`
- **Description**: Attendee ratings and evaluations for completed events.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `eventId`: `BIGINT`, Not Null.
  - `participantId`: `VARCHAR(50)`, Not Null.
  - `rating`: `INT`, Not Null, Validated: 1 <= rating <= 5.
  - `comments`: `VARCHAR(500)`, Nullable.
  - `submittedAt`: `DATETIME`, Not Null.
- **Constraints & Indexes**:
  - Unique Constraint: `uk_feedback_event_participant` on (`eventId`, `participantId`).
  - Index: `idx_feedback_event` on (`eventId`).

---

### 2.7 `notification-service` (Schema: `eventsphere_notification_db`)

#### Entity: `Notification`
- **Description**: Outbox table managing asynchronous participant notifications.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `recipientUserId`: `VARCHAR(50)`, Not Null.
  - `recipientEmail`: `VARCHAR(120)`, Not Null.
  - `eventId`: `BIGINT`, Not Null.
  - `notificationType`: `VARCHAR(30)` (Enum: `NotificationType`), Not Null.
  - `subject`: `VARCHAR(150)`, Not Null.
  - `content`: `TEXT`, Not Null.
  - `status`: `VARCHAR(20)` (Enum: `NotificationStatus`), Not Null, Default: `'PENDING'`.
  - `sentAt`: `DATETIME`, Nullable.
  - `retryCount`: `INT`, Not Null, Default: 0.

---

### 2.8 `analytics-service` (Schema: `eventsphere_analytics_db`)

#### Entity: `EventAnalyticsSnapshot`
- **Description**: Real-time aggregated metrics per event for fast reporting queries.
- **Fields**:
  - `id`: `BIGINT`, Primary Key, Auto-Increment.
  - `eventId`: `BIGINT`, Not Null, Unique.
  - `totalCapacity`: `INT`, Not Null.
  - `totalRegistered`: `INT`, Not Null, Default: 0.
  - `totalAttended`: `INT`, Not Null, Default: 0.
  - `totalAbsent`: `INT`, Not Null, Default: 0.
  - `feedbackCount`: `INT`, Not Null, Default: 0.
  - `averageRating`: `DECIMAL(3,2)`, Not Null, Default: 0.00.
  - `lastCalculatedAt`: `DATETIME`, Not Null.

---

## 3. Sample Seed Data

### 3.1 Venues & Rooms (`venue-service`)
**Table: `venues`**
| id | name | address | city |
|---|---|---|---|
| 1 | Global Tech Campus - Block A | 100 Innovation Way | Bengaluru |
| 2 | Executive Learning Center | 45 Leadership Blvd | Hyderabad |

**Table: `rooms`**
| id | venueId | roomNumber | maxSeatingCapacity | amenities |
|---|---|---|---|---|
| 1 | 1 | Auditorium Alpha | 200 | Dual 4K Projectors, Stage AV, Surround Sound |
| 2 | 1 | Training Lab 102 | 30 | 30 Developer Workstations, Smartboard |
| 3 | 2 | Executive Suite 4 | 50 | Video Conferencing Bar, Executive Seating |

### 3.2 Events (`event-service`)
**Table: `events`**
| id | title | eventType | status | startTime | endTime | venueId | roomId | eligibleDepartment | maxCapacity | createdBy |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Cloud Architecture Summit | CORPORATE | PUBLISHED | 2026-11-10 09:00:00 | 2026-11-10 17:00:00 | 1 | 1 | Engineering | 150 | org_smith |
| 2 | AI & Machine Learning Workshop | EDUCATIONAL | PUBLISHED | 2026-11-15 10:00:00 | 2026-11-15 16:00:00 | 1 | 2 | ALL | 30 | org_smith |
| 3 | Annual Leadership Forum | CORPORATE | DRAFT | 2026-12-01 13:00:00 | 2026-12-01 18:00:00 | 2 | 3 | Management | 45 | org_jones |

### 3.3 Speakers (`speaker-service`)
**Table: `speakers`**
| id | fullName | email | expertiseArea |
|---|---|---|---|
| 1 | Dr. Arvind Rao | a.rao@example.com | Cloud Distributed Systems |
| 2 | Maya Chen | m.chen@example.com | Generative AI & Deep Learning |

### 3.4 Registrations (`registration-service`)
**Table: `registrations`**
| id | eventId | participantId | participantEmail | participantDepartment | status | registeredAt |
|---|---|---|---|---|---|---|
| 1 | 1 | usr_dev101 | dev101@company.com | Engineering | CONFIRMED | 2026-10-28 08:00:00 |
| 2 | 1 | usr_dev102 | dev102@company.com | Engineering | CONFIRMED | 2026-10-28 08:05:00 |
| 3 | 2 | usr_qa201 | qa201@company.com | QA | CONFIRMED | 2026-10-28 08:15:00 |

### 3.5 Attendance (`attendance-service`)
**Table: `attendance_records`**
| id | eventId | participantId | status | checkInTime | verifiedBy |
|---|---|---|---|---|---|
| 1 | 1 | usr_dev101 | ATTENDED | 2026-11-10 08:55:00 | org_smith |
| 2 | 1 | usr_dev102 | ABSENT | NULL | org_smith |

### 3.6 Feedback (`feedback-service`)
**Table: `feedback`**
| id | eventId | participantId | rating | comments | submittedAt |
|---|---|---|---|---|---|
| 1 | 1 | usr_dev101 | 5 | Outstanding session on microservices resilience patterns! | 2026-11-10 17:30:00 |

---

## 4. Requirement-to-Entity Mapping

| Functional Requirement | Target Entity | Key Fields Leveraged | Enforced Constraints |
|---|---|---|---|
| **FR-EV-01, FR-EV-02** | `Event` | `title`, `eventType`, `status`, `maxCapacity` | Non-null, string lengths, valid Enum |
| **FR-EV-03** | `EventApproval` | `eventId`, `adminId`, `decision`, `remarks` | FK to Event, timestamp |
| **FR-VN-01, FR-VN-02** | `Room`, `VenueBookingSlot` | `roomId`, `startTime`, `endTime`, `bookingStatus` | Collision window index, optimistic lock |
| **FR-SP-01, FR-SP-02** | `Speaker`, `SpeakerAssignment` | `speakerId`, `eventId`, `sessionTitle` | Valid email, FK references |
| **FR-RG-01, FR-RG-02** | `ParticipantProfile` | `userId`, `department` | Unique user, department comparison |
| **FR-RG-03, FR-RG-04** | `Registration` | `eventId`, `participantId`, `status` | Unique composite `(eventId, participantId, status)` |
| **FR-AT-01** | `AttendanceRecord` | `eventId`, `participantId`, `status` | Unique `(eventId, participantId)` |
| **FR-FB-01, FR-FB-02** | `Feedback` | `eventId`, `participantId`, `rating` | Unique `(eventId, participantId)`, Rating 1–5 |
| **FR-NT-01** | `Notification` | `recipientEmail`, `notificationType`, `status` | Outbox pattern |
| **FR-AN-01, FR-AN-02** | `EventAnalyticsSnapshot` | `totalRegistered`, `totalAttended`, `averageRating` | Materialized aggregate snapshot |

---

## 5. JPA Mapping Guidelines & Conventions

1. **Entity Identification**:
   - Primary key generated via `@Id @GeneratedValue(strategy = GenerationType.IDENTITY)`.
2. **Enumeration Storage**:
   - Always persist Enums as readable strings: `@Enumerated(EnumType.STRING)`. Never use ordinal integers.
3. **Optimistic Locking**:
   - Entities subject to high-concurrency state transitions (`Event`, `VenueBookingSlot`) must annotate an integer version property with `@Version`.
4. **Audit Timestamps**:
   - `@CreationTimestamp` on `createdAt` / `registeredAt`.
   - `@UpdateTimestamp` on `updatedAt`.
5. **Table & Column Naming**:
   - Use snake_case in MySQL table and column definitions: `@Table(name = "events")`, `@Column(name = "start_time", nullable = false)`.
6. **Unique Constraints**:
   - Declare composite unique indexes explicitly via `@Table(uniqueConstraints = @UniqueConstraint(name = "uk_name", columnNames = {"col1", "col2"}))`.
