# Raw Requirements: Infy_EventSphere Enterprise Event Management Platform

## 1. Business Objective
Infy_EventSphere is an enterprise event management platform designed to support the complete lifecycle of corporate, educational, and public events—from planning and registration to execution and feedback analysis.

The primary objectives are:
- Improve event planning efficiency and workflow automation across diverse event types.
- Enhance attendee engagement before, during, and after events.
- Establish operational transparency across resource allocation, approvals, and compliance.
- Empower decision-making through centralized event data and real-time operational reporting.
- Lay the foundation for a modular, microservices-aligned architecture managing distinct capabilities independently.

---

## 2. Users and Actors

### 2.1 Event Organizers
- Create and configure events, schedules, agendas, and session tracks.
- Allocate and manage venues and physical/virtual rooms.
- Coordinate speakers and manage session materials.
- Monitor registrations and real-time attendee lists.
- Track attendee participation and attendance during execution.

### 2.2 Participants / Attendees
- Discover and browse upcoming corporate, educational, and public events.
- Register online for eligible events and specific sessions.
- Receive timely notifications, confirmations, and reminders.
- Access event resources, slides, agendas, and materials.
- Submit post-event feedback, ratings, and survey evaluations.

### 2.3 Administrators
- Oversee and manage the event approval workflow.
- Monitor venue utilization, capacity thresholds, and resource efficiency.
- Enforce compliance, governance, and organizational policies.
- Access centralized operational reporting and executive analytics dashboards.

### 2.4 Speakers / Presenters (Secondary Actor)
- View assigned session schedules and agendas.
- Share presentation resources and session prerequisites with attendees.

### 2.5 System / Notification Service (Automated Actor)
- Dispatches automated confirmations, schedule change alerts, reminders, and feedback links.

---

## 3. Key Features

### 3.1 Event Planning & Lifecycle Management
- Creation of events categorized by type (Corporate, Educational, Public).
- Definition of event details: title, description, agendas, session tracks, target audience, and dates/times.
- Lifecycle state transitions: Draft, Pending Approval, Approved, Published, In Progress, Completed, Cancelled.

### 3.2 Venue Allocation & Scheduling
- Venue inventory management with capacity limits, amenities, and location details.
- Scheduling engine with collision detection to prevent double-booking of venues and time slots.
- Tracking of venue utilization metrics across departments and dates.

### 3.3 Speaker Coordination
- Profiling of speakers and session assignments.
- Centralized scheduling to prevent speaker session clashes.
- Association of presentation materials and speaker bios with events.

### 3.4 Discovery & Registration Management
- Searchable, filterable event catalog for prospective participants.
- Eligibility verification prior to registration.
- Capacity-constrained online registration workflow.
- Strict duplicate registration prevention.
- Registration confirmation with clear status outcomes.

### 3.5 Automated Notifications & Communications
- Triggered notifications for registration confirmations, cancellations, and waitlist movements.
- Broadcast alerts for venue changes, schedule adjustments, or speaker updates.
- Post-event reminders prompting attendees to complete feedback forms.

### 3.6 Attendance Tracking & Execution
- Check-in tracking for attendees during live events.
- Session-level and overall event attendance records.
- Identification of no-shows and real-time participation metrics.

### 3.7 Post-Event Feedback & Analysis
- Standardized feedback forms and rating mechanisms for completed events.
- Aggregation of attendee ratings, comments, and satisfaction indices.
- Feedback access restricted to confirmed attendees.

### 3.8 Administration, Compliance & Real-Time Reporting
- Centralized approval desk for administrators to approve/reject proposed events.
- Compliance checklist validation for enterprise policies.
- Real-time reporting on registrations, attendance rates, venue utilization, and participant demographics.

---

## 4. Business Rules

### BR-01: Event Approval Requirement
- An event cannot be published or opened for participant registration without explicit administrative approval.

### BR-02: Venue Conflict Prevention
- A venue or room cannot be assigned to more than one event or session during the same time window.

### BR-03: Participant Eligibility
- Only eligible participants meeting the defined criteria (e.g., department, role, prerequisites) are permitted to register for restricted corporate or educational events. Public events are open to all registered users.

### BR-04: Duplicate Registration Prevention
- A participant cannot register more than once for the same event or overlapping sessions within an event.

### BR-05: Capacity Enforcement
- Once an event reaches its maximum participant capacity, further registrations must be locked or placed in a waitlist queue (if enabled).

### BR-06: Feedback Eligibility
- Feedback and ratings can only be submitted by participants who were marked as attended for the event, and only after the event has reached `Completed` status.

### BR-07: Lifecycle Consistency
- Completed or Cancelled events cannot accept new registrations, schedule changes, or venue modifications.

---

## 5. Assumptions

1. **Identity & Authentication**: The platform operates in an enterprise environment where user identity, roles (Organizer, Participant, Admin), and departmental data are authenticated through an enterprise directory or identity provider.
2. **Modular Microservice Decomposition**: The business capabilities (Events, Venues, Registrations, Notifications, Attendance, Feedback, Reporting) will be mapped to decoupled microservice boundaries.
3. **Database Availability**: A relational or multi-model persistence store will be utilized to persist state transitions, relational mappings (venues, events, registrations), and ensure transactional consistency.
4. **Communication Channels**: Standard notification channels (e.g., email / in-app notifications) are sufficient for participant alerts and reminders.
5. **Sample Demo Data**: Demo environments will be pre-seeded with representative events, venues, participants, and administrators across corporate, educational, and public event types.

---

## 6. Open Questions

1. **Eligibility Rules Specification**: What specific attributes dictate participant eligibility (e.g., job band, department code, training prerequisite completion, manager approval)?
2. **Registration Approval Model**: Is registration strictly self-service upon meeting eligibility and capacity, or does certain corporate/educational training require manager or organizer approval?
3. **Waitlist Handling**: If an event reaches capacity, should an automated waitlist queue be maintained with auto-promotion upon cancellation?
4. **Attendance Verification Mechanism**: Will event attendance be recorded via manual organizer check-in, self check-in, or digital badge/QR code scanning?
5. **Compliance Requirements**: What specific enterprise compliance verifications (e.g., safety limits, mandatory disclosures, privacy policies) must administrators confirm before event approval?
6. **Commercial/Fee Scope**: Are public events strictly free of charge or sponsored, or will payment gateway integrations be needed in a future phase?
