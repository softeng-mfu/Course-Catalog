# Course Catalog Platform — Architecture Design

## 1. Minimum Data Model

| Entity | Essential fields | Relationships | Who may create / read / update / delete? |
|---|---|---|---|
| User | user_id, email_or_username, password_hash, role | An administrator performs catalog-management actions. | Admin: own account read/update as supported. Student: no account required in MVP. |
| Course | course_id, course_code, course_name, description, status, created_at, updated_at | Many-to-many with Instructor; zero or more prerequisite Courses. | Admin: create, read, update, delete/publish. Student: read published records only. |
| Instructor | instructor_id, name, email | Many-to-many with Course through CourseInstructor. | Admin: create, read, update, delete. Student: read approved public fields only. |
| CourseInstructor | course_id, instructor_id | Join table between Course and Instructor. | Admin: manage through course/instructor operations. |
| CoursePrerequisite | course_id, prerequisite_course_id | Self-referential Course relationship. | Admin: create and delete. Student: read only. |

The system does not store student grades, transcripts, enrollment history, payment data, or unnecessary AI conversation history.

### Data Integrity Rules

| Rule | Backend control | Database control |
|---|---|---|
| Course code is unique | Validate and return a clear conflict error. | UNIQUE constraint on course_code. |
| Prerequisite references exist | Verify both course records exist. | Foreign keys to Course. |
| A course cannot require itself | Reject matching course IDs. | Backend validation. |
| A prerequisite pair cannot be duplicated | Check before insert. | UNIQUE(course_id, prerequisite_course_id). |
| Prerequisites cannot form a cycle | Traverse the prerequisite graph before saving. | Persist only after successful transactional validation. |
| Course deletion is safe | Reject deletion or remove permitted dependent links in a transaction. | Foreign-key deletion policy prevents orphaned data. |

PostgreSQL is the final protection against concurrent writes. If two administrators create the same course code, one request succeeds and the other receives a conflict from the unique constraint.

## 2. Platform Architecture

~~~text
Student / Advisor / Admin
          │
          │ HTTPS
          ▼
React Frontend
  ├── Course Search / Catalog
  ├── Course Details
  ├── Course / Prerequisite Assistance
  └── Protected Admin Management
          │
          │ JSON over REST
          ▼
Spring Boot Backend (modular monolith)
  ├── Course, Instructor, and Prerequisite APIs
  ├── Authorization checks for admin actions
  ├── AI-assisted explanation + deterministic fallback
  └── Webhook publisher
          │
          │ JDBC / SQL
          ▼
Supabase PostgreSQL
  ├── users
  ├── courses
  ├── instructors
  ├── course_instructors
  ├── course_prerequisites
  └── webhook_events

Supabase Auth
  └── Admin sign-in and role verification

Spring Boot ── HTTPS ──> Optional AI Provider
Spring Boot ── HTTPS ──> Inter-team Webhook Receiver
~~~

### Component Rationale

| Component | Data moving through it | Why it is needed |
|---|---|---|
| React frontend | Search terms, public catalog responses, authenticated admin requests. | Presents the three PRD screens and admin interface. |
| Spring Boot backend | REST requests/responses, authenticated write requests, webhook payloads. | Centralizes business rules, validation, security, AI fallback, and integration logic. |
| PostgreSQL | Users, courses, instructors, and course relationships. | Stores relational catalog data and enforces integrity. |
| AI provider | Catalog-grounded context and an optional explanation. | Supports the AI-assisted requirement without becoming the source of truth. |
| Webhook receiver | A course.updated event after a relevant catalog change. | Provides the required inter-team integration. |

## 3. Architecture Decision Questions

| Question | Decision |
|---|---|
| Where does the client run? | In the user's browser as a React application. |
| Where does business logic run? | In Spring Boot. React does not decide authorization or data integrity. |
| Where is each type of data stored? | PostgreSQL stores all required structured data. No file storage is needed for this MVP. |
| Where does authentication happen? | Supabase Auth authenticates administrators and verifies role membership before protected actions. |
| Where is authorization enforced? | The Spring Boot API validates the authenticated user and checks admin permissions on every create, update, delete, and prerequisite-management endpoint. |
| Where are files stored? | Nowhere; file upload is outside the MVP scope. |
| Which external services are called, from where, and with what failure plan? | Spring Boot calls the optional AI provider and inter-team webhook receiver. AI failure uses deterministic catalog data; webhook failure is logged for retry. |
| Which component prevents duplicate or conflicting operations? | Spring Boot validates requests; PostgreSQL unique constraints, foreign keys, and transactions provide final enforcement. |

## 4. Core User Journey Architecture

| Step | User action | System response | Data / rule involved | Possible failure |
|---:|---|---|---|---|
| 1 | Student searches or browses courses. | React calls GET /api/courses; Spring Boot returns matching published courses. | Published status; validated query parameters. | Network failure, invalid query, or no results. |
| 2 | Student opens a course. | React calls GET /api/courses/{id}; backend returns course, instructors, and prerequisites. | Course and relationship data; public-data filtering. | Missing course returns 404. |
| 3 | Student requests prerequisite help. | Backend returns catalog-grounded AI assistance or a deterministic fallback. | AI grounding and fallback rules. | AI timeout or unavailable provider. |
| 4 | Admin changes catalog information. | Backend authenticates, authorizes, validates, and writes transactionally. | ADMIN role, integrity rules, database constraints. | Invalid input, 401/403, duplicate code, or circular prerequisite. |
| 5 | Relevant course change completes. | Backend attempts to publish course.updated. | Webhook event and idempotency key. | Delivery is logged and retried; catalog update remains committed. |

## 5. API Boundaries

| Method | Endpoint | Purpose | Access |
|---|---|---|---|
| GET | /api/courses | Search/list published courses. | Public |
| GET | /api/courses/{id} | Get one course with details. | Public |
| POST | /api/courses | Create a course. | Admin |
| PUT | /api/courses/{id} | Update a course. | Admin |
| DELETE | /api/courses/{id} | Delete a course safely. | Admin |
| GET | /api/instructors/{id} | Get an instructor and associated courses. | Public |
| GET | /api/courses/{id}/prerequisites | Get a course's prerequisite courses. | Public |
| POST | /api/courses/{id}/assistance | Get an AI or deterministic explanation. | Public |

Responses are JSON. The API returns 400 for invalid input, 404 for a missing record, 401/403 for unauthorized changes, 409 for duplicate data, and 422 for invalid prerequisite relationships.

## 6. Failure-Path Design

| Failure | Architecture response |
|---|---|
| Network request fails or repeats | React gives retry feedback; read requests are safe to retry. |
| Invalid or malicious input | Spring Boot rejects it before persistence with a controlled error. |
| Student calls an admin API directly | Backend returns 401 or 403; no change is made. |
| Two admins create the same course | PostgreSQL unique constraint ensures only one succeeds. |
| Self, duplicate, or circular prerequisite | Backend rejects before write; database also blocks duplicate pairs and invalid references. |
| PostgreSQL fails during a write | The transaction rolls back; backend returns controlled server error without database details. |
| AI provider fails | Backend returns deterministic information from stored course and prerequisite fields. |
| Webhook delivery fails | The catalog update remains committed; failure is logged and retried with an idempotency key. |

## 7. Deployment Architecture

| Layer | Deployment choice | Reason |
|---|---|---|
| Frontend | Vercel for the React frontend. | Supports a fast, low-cost deployment and preview environments. |
| Backend | Spring Boot API deployed on a small Java-compatible host or serverless-compatible backend layer. | Keeps the API logic and security centralized while staying realistic for the project. |
| Database | Supabase PostgreSQL for persistence. | Provides a managed relational database with low operational burden. |
| Authentication | Supabase Auth for administrator login and role checks. | Simplifies secure admin access for a small team. |
| Configuration | Environment variables for DB credentials, auth settings, AI keys, and webhook secrets. | Keeps secrets outside source control. |

Use HTTPS for all browser-to-backend and backend-to-external-service traffic. Development runs React, Spring Boot, and PostgreSQL locally with automated tests before merge, while production uses Vercel + Supabase as the deployment target.

## 8. Deliberately Excluded Complexity

- No microservices: the catalog is one small domain for a 3–5 person team.
- No API gateway or service mesh: there is one backend service.
- No message broker: logged retries and idempotency satisfy the single webhook requirement.
- No file storage: uploads are outside the MVP.
- No separate AI database: PostgreSQL catalog data remains the source of truth.

## 9. Architecture Verification Checklist

- [ ] Every component supports a PRD requirement.
- [ ] React, Spring Boot, and PostgreSQL match PRD.md.
- [ ] The backend, not the frontend, enforces authorization and business rules.
- [ ] PostgreSQL constraints protect course and prerequisite integrity.
- [ ] Public APIs expose only published data.
- [ ] AI assistance is catalog-grounded and has deterministic fallback.
- [ ] Webhook failures cannot corrupt or reverse catalog updates.
- [ ] The design is feasible for 3–5 developers, one semester, and a 0 THB deployment budget.
