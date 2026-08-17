# Product Requirements Document

## 1. Product Overview

### Product Name

Course Catalog Platform

### Problem Statement

University course information is often scattered across websites, documents, and emails. Students spend too much time finding course details, instructors, and prerequisites, and can make planning decisions using incomplete or outdated information. This product provides one searchable, reliable place to explore published course information. It is a discovery and information platform, not a course-registration system.

### Target Users

- **Students (primary):** search and browse courses, review instructors and prerequisites, and decide whether a course is relevant.
- **Course administrators (secondary):** maintain course, instructor, and prerequisite data.

### Product Goal

Deliver a small, zero-budget web MVP that centralizes course information, makes prerequisite relationships understandable, and lets authorized administrators maintain accurate published catalog data.

---

## 2. Scope

### In Scope

- Search and browse published courses by code or name.
- Course-detail pages showing description, instructors, and prerequisites.
- Instructor information associated with courses.
- Administrator authentication and CRUD management of courses and instructors.
- Administrator management of prerequisite relationships.
- At least six documented REST operations.
- One `course.updated` catalog webhook.
- AI-assisted explanation of course/prerequisite information, grounded only in catalog data.
- Deterministic prerequisite lookup when AI is unavailable or invalid.
- Automated tests for core API and validation paths.

### Out of Scope

- Course registration, enrollment, tuition, payments, attendance, grades, transcripts, or student records.
- Real-time university SIS integration, seat availability, timetable optimization, or degree auditing.
- Mobile application, chat, social features, advanced analytics, and complex recommendations.
- Microservices, API gateways, message brokers, Kubernetes, and multi-university support.

---

## 3. User Roles

### User

Students can read published catalog content. They can search and browse courses, open course details, view instructor and prerequisite information, and request an AI-assisted explanation. They cannot change catalog data.

### Admin

Course administrators authenticate before creating, updating, or deleting courses and instructors, and before managing prerequisite relationships. They can view webhook delivery failures for retry.

---

## 4. User Journey

### Main Journey

1. A student opens the Course Catalog and searches or browses published courses.
2. The student selects a course from the results.
3. The student reviews its code, description, instructors, and prerequisite courses.
4. The student opens the supporting course/prerequisite-assistance view when clarification is needed; it returns a catalog-grounded AI explanation or the stored prerequisite data as fallback.

The MVP has three core screens: Course Search/Catalog, Course Details, and Course/Prerequisite Assistance.

---

## 5. Functional Requirements

### FR-01 Search and browse courses

The system shall allow students to search published courses by course code or course name and browse available courses.

### FR-02 View course details

The system shall display a selected course's code, name, description, associated instructor(s), and prerequisite(s).

### FR-03 Manage courses

An authenticated administrator shall be able to create, view, update, and delete course records.

### FR-04 Manage instructors

Students shall be able to view instructors associated with a course. An authenticated administrator shall be able to create, update, and remove instructor records and course associations.

### FR-05 Manage prerequisites

Students shall be able to view prerequisite courses. An authenticated administrator shall be able to create and remove prerequisite relationships subject to validation rules.

### FR-06 Admin authentication

The system shall require administrator authentication and backend authorization for every catalog-management operation.

### FR-07 REST API

The system shall expose and document at least six REST operations for course search, course retrieval, course CRUD, instructor retrieval, and prerequisite retrieval.

### FR-08 Catalog webhook

The system shall publish a `course.updated` webhook when relevant catalog information changes, without allowing delivery failure to roll back the catalog update.

### FR-09 AI-assisted explanation

The system shall use existing course and prerequisite data to provide an AI-assisted explanation of a student's course question. It shall not invent catalog facts.

### FR-10 Deterministic fallback

If the AI request fails, times out, or returns unusable output, the system shall return a deterministic explanation constructed from stored course and prerequisite data.

---

## 6. Non-Functional Requirements

### NFR-01 Performance

Normal course searches should return within 2 seconds and public pages should load within 3 seconds under expected student-project usage.

### NFR-02 Security

Only authorized administrators may modify catalog data. Passwords must never be stored in plaintext, API input must be validated, and public endpoints must not expose unpublished or sensitive data.

### NFR-03 Availability

The application shall remain usable under normal project workloads. Core course discovery and stored prerequisite information must remain available when the optional AI provider is unavailable.

### NFR-04 Cost

The MVP shall be deployable with a 0 THB budget using free-tier services where necessary.

### NFR-05 Maintainability and testability

The codebase shall be understandable by a team of 3–5 student developers and include automated coverage for essential API, authorization, data-integrity, webhook, and fallback paths.

---

## 7. Business Rules

### BR-01 Catalog integrity

Every course must have a unique course code. Each prerequisite must reference an existing course; a course cannot be its own prerequisite; duplicate prerequisite pairs are prohibited; and prerequisite relationships must not form a circular dependency.

### BR-02 Access and publication

Students can read only published course information. Only authorized administrators can modify catalog data. Course changes must be validated before becoming publicly available.

### BR-03 AI grounding and fallback

AI-generated explanations must be grounded in available catalog data. A deterministic catalog-based response is required whenever AI assistance is unavailable or invalid.

### BR-04 Deletion safety

Deleting a course must not leave invalid prerequisite or instructor relationships; the operation must be rejected or safely remove related records according to database referential rules.

---

## 8. Data Model

### User

Fields: `user_id`, `email_or_username`, `password_hash`, `role` (`ADMIN`).

Only administrators need accounts in the MVP. Credentials and password hashes are sensitive; plaintext passwords must not be stored.

### Entity A: Course

Fields: `course_id`, `course_code`, `course_name`, `description`, `status` (published/unpublished), `created_at`, `updated_at`.

A course can have many instructors through `CourseInstructor` and zero or more prerequisite courses through `CoursePrerequisite`. Administrators manage records; students read published records only.

### Entity B: Instructor

Fields: `instructor_id`, `name`, `email` (internal or public only with approval).

Instructors have a many-to-many relationship with courses through `CourseInstructor`. Administrators manage records; students read approved public information only.

### Relationship entities and constraints

- `CourseInstructor(course_id, instructor_id)` joins courses and instructors.
- `CoursePrerequisite(course_id, prerequisite_course_id)` models a course dependency; both fields reference `Course`.
- `CoursePrerequisite` has a composite unique constraint on `(course_id, prerequisite_course_id)`.
- `course_code` is unique; required published fields are non-null; all relationship keys are foreign keys.

The system must not store student grades, transcripts, enrollment or registration history, payment information, unnecessary AI conversation history, or unverified personal data.

---

## 9. Architecture

### Architecture Overview

The product is a simple modular monolith: a React web client calls a Spring Boot REST API, which applies validation, authentication, authorization, AI fallback logic, and webhook delivery before persisting catalog data in PostgreSQL.

### Architecture Diagram

```text
React Frontend
  ├─ Course Search / Details / Assistance
  └─ Admin Management
             │ HTTPS / REST
             ▼
Spring Boot Backend
  ├─ Course, Instructor, Prerequisite APIs
  ├─ Authentication / Authorization
  ├─ AI Assistance + deterministic fallback
  └─ Webhook Publisher
             │ JPA / SQL
             ▼
PostgreSQL: Users, Courses, Instructors, CourseInstructor, CoursePrerequisite
```

### Components

#### Frontend

React renders the three student screens and protected administrator management views.

#### Backend

Spring Boot provides REST endpoints, business-rule validation, authorization, webhook publishing, and AI-assisted explanation orchestration.

#### Database

PostgreSQL stores the normalized catalog and enforces unique and foreign-key constraints.

#### Authentication

Spring Boot authenticates administrators and enforces authorization server-side; frontend role checks alone are not security.

#### Storage

No separate file storage is required for the MVP.

#### External Services

An optional AI provider creates catalog-grounded explanations; an inter-team webhook receiver consumes `course.updated` events.

---

## 10. Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Frontend | React | Suitable for a small catalog UI and fast iteration. |
| Backend | Spring Boot | Provides clear REST, security, and validation layers. |
| Database | PostgreSQL | Required relational store for course relationships and constraints. |
| Auth | Supabase Auth | Simplifies secure admin login and role checks. |
| Hosting | Vercel + Supabase | Low cost, free-tier friendly, and suitable for the MVP. |

---

## 11. API / Interfaces

### API-01 Course APIs

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| GET | `/api/courses` | Search/list published courses | Public |
| GET | `/api/courses/{id}` | Get course details | Public |
| POST | `/api/courses` | Create a course | Admin |
| PUT | `/api/courses/{id}` | Update a course | Admin |
| DELETE | `/api/courses/{id}` | Delete a course safely | Admin |

### API-02 Supporting APIs

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| GET | `/api/instructors/{id}` | Get instructor and associated courses | Public |
| GET | `/api/courses/{id}/prerequisites` | Get a course's prerequisites | Public |
| POST | `/api/courses/{id}/assistance` | Get AI or fallback explanation | Public |

Responses use JSON. Errors return controlled status codes and an error object, for example `{ "code": "VALIDATION_ERROR", "message": "Course code is required." }`.

The webhook payload includes `event`, `courseId`, `courseCode`, `timestamp`, and a payload version. Delivery uses an idempotency key, logs failures, and can be retried safely.

---

## 12. Security

### Authentication

Administrator login is required for management endpoints. Passwords are hashed using an approved password-hashing algorithm and never returned by the API.

### Authorization

Spring Boot enforces admin authorization on every create, update, delete, prerequisite-management, and webhook-retry endpoint. Role data supplied by the frontend is not trusted.

### Data Protection

Public endpoints return published, approved catalog fields only. Internal instructor contact information and administrative data are not exposed. Input is validated and database errors are not revealed to users.

---

## 13. Error Handling

### Expected Errors

- Invalid query parameters or required fields: `400 Bad Request`.
- Missing course or instructor: `404 Not Found`.
- Unauthenticated or unauthorized modification: `401` or `403`.
- Duplicate course code or prerequisite relationship: `409 Conflict`.
- Self-referential or circular prerequisite: `422 Unprocessable Entity`.

### Failure Scenarios

- If the database fails during a write, the API returns a controlled server error without database internals.
- If a webhook fails after a successful catalog update, the update remains valid; the failure is logged for safe retry.
- If AI times out, fails, or contradicts available data, the assistance endpoint returns the deterministic catalog-based fallback.

---

## 14. Deployment

### Development

Develop locally with React, Spring Boot, and PostgreSQL; use a shared Git repository and environment-specific configuration. Automated tests run before merging changes.

### Production

Deploy the React frontend to Vercel and use Supabase for PostgreSQL and admin authentication. Configure HTTPS, environment secrets, administrator bootstrap, and the downstream webhook URL. No paid infrastructure, microservices, or orchestration platform is required.

---

## 15. Constraints

- Budget: 0 THB deployment budget.
- Time: one semester.
- Team: 3–5 student developers.
- Required stack: React, Spring Boot, and PostgreSQL.
- Prefer free-tier services and simple modular-monolith architecture.
- Avoid unnecessary enterprise infrastructure and live university-system integration.

---

## 16. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| AI output is inaccurate or unavailable | Students receive misleading or no explanation | Ground output in catalog data, validate it, and use deterministic fallback. |
| Duplicate or circular prerequisites | Incorrect course-planning information | Enforce database uniqueness and backend graph validation. |
| Unauthorized API calls | Catalog data is modified improperly | Enforce server-side authentication and authorization. |
| Concurrent duplicate course creation | Inconsistent catalog | Enforce a database-level unique `course_code` constraint. |
| Webhook delivery fails | Downstream catalog data becomes stale | Log failure, use idempotency keys, and retry safely. |
| Free-tier limits | Service instability or deployment limits | Keep the MVP small and select compatible free-tier providers early. |

---

## 17. Acceptance Criteria

### MVP is complete when:

- [ ] A student can search/browse published courses and open a course-detail page.
- [ ] Course details display code, name, description, instructor(s), and prerequisite(s).
- [ ] An authorized administrator can create, update, and delete course data and manage instructors/prerequisites.
- [ ] Unauthorized users cannot perform catalog-management operations.
- [ ] At least six documented REST operations work with controlled error responses.
- [ ] The system emits the defined `course.updated` webhook for relevant catalog changes.
- [ ] AI assistance is catalog-grounded and failure produces deterministic stored-course information.
- [ ] At least seven automated tests pass, including search, course retrieval, duplicate-code rejection, prerequisite validation, authorization, duplicate-prerequisite rejection, and AI fallback.
- [ ] The application can be deployed using the required stack and a zero-budget setup.

---

## 18. Future Improvements

- Add instructor self-service and department approval workflows.
- Integrate trusted university SIS/LMS data sources.
- Provide prerequisite-graph visualization and richer course planning.
- Add seat availability, schedules, recommendations, and mobile support only after validating demand.
