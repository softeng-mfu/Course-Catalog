# Platform Architecture Diagram

```text
Student / Advisor / Admin
          │
          │ HTTPS
          ▼
React Frontend
  ├── Course Search / Catalog Screen
  ├── Course Details Screen
  ├── Course / Prerequisite Assistance Screen
  └── Protected Admin Management UI
          │
          │ REST API / JSON
          ▼
Spring Boot Backend
  ├── Course API
  ├── Instructor API
  ├── Prerequisite API
  ├── Authorization checks for admin actions
  ├── AI-assisted explanation service
  ├── Deterministic fallback logic
  └── Webhook publisher
          │
          │ JPA / SQL
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
Spring Boot ── HTTPS ──> Downstream Inter-team Webhook Receiver
```

## Why this architecture fits the project

- React provides the user-facing catalog UI and admin screens.
- Spring Boot centralizes business rules, validation, authorization, and integration logic.
- Supabase PostgreSQL stores the course catalog data with relational integrity.
- Supabase Auth secures admin-only actions without requiring custom auth infrastructure.
- Vercel can host the frontend, while the backend and database remain lightweight and low-cost.
- The AI and webhook integrations remain optional and isolated, with deterministic fallback for failure conditions.

## Deployment model

```text
Vercel
  └── React frontend

Supabase
  ├── PostgreSQL database
  ├── Auth service
  └── optional storage

Spring Boot API
  └── deployed on a simple Java-compatible host or backend runtime
```

## Key design decisions

- The frontend is not trusted for authorization or data integrity.
- The backend remains the single source of business logic.
- Database constraints enforce unique course codes and prerequisite integrity.
- AI-generated content is not treated as the system of record.
- Webhook failures do not roll back successful catalog updates.
