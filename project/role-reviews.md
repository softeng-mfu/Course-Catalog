## Quality and Security Review

### Summary

The PRD is strong on product goals, user roles, and core security intent, but it still leaves several quality and security decisions underspecified. The most important gaps are the absence of a concrete test strategy for each acceptance criterion, insufficiently explicit server-side authorization rules for every admin action, and incomplete guidance for safe handling of secrets, validation, and failure modes. These issues matter because the product deals with catalog integrity, admin-only writes, and AI fallback logic, all of which must be protected from misuse and partial failure.

### Findings

| ID | PRD section | Issue or question | Why it matters | Severity | Recommended change |
|---|---|---|---|---|---|
| QS-01 | Acceptance Criteria, Functional Requirements, Error Handlin

g | The PRD lists acceptance criteria and expected errors, but it does not define how each acceptance criterion will be tested in practice. | A team cannot confirm the MVP is complete without explicit test coverage for success, validation, authorization, duplicate data, and fallback behavior. This is especially important for course uniqueness, prerequisite cycles, and AI fallback. | High | Add a test matrix mapping each acceptance criterion to API or UI tests, including expected request/response behavior, input validation, and failure conditions. |
| QS-02 | Security, Authentication, Authorization | The PRD says admin authentication is required, but it does not fully define which endpoints must be protected and which checks are enforced at the backend versus the frontend. | Interface-only checks are not enough. A user could bypass a UI restriction and still execute restricted operations if the backend does not verify role and ownership on every mutation. | Critical | State explicitly that every create, update, delete, prerequisite-management, and webhook-retry endpoint must enforce server-side authentication and admin authorization using trusted session or token claims, not frontend role flags. |
| QS-03 | User Roles, Data Model, Security | The PRD states that students can read only published courses and admins can manage data, but it does not define how access is scoped for unpublished items, instructor records, or admin-only operations beyond general role checks. | Without a clear permission model, a developer may accidentally expose unpublished or sensitive data, or over-permit administrator actions. | High | Define a permission matrix that spells out which roles may read, create, update, or delete each entity, including unpublished records, private instructor contact fields, and webhook logs. |
| QS-04 | Security, Data Protection, Authentication | Sensitive data handling is mentioned, but the PRD does not specify the exact protection mechanisms for passwords, secrets, and API credentials. | Password hashes and external credentials must be protected from exposure, logs, and accidental storage. Missing protections create a direct security risk. | High | Require password hashing with a modern algorithm such as bcrypt or Argon2, prohibit plaintext password storage, keep secrets in environment variables or secret managers, and ensure logs do not contain credentials, tokens, or raw request payloads. |
| QS-05 | Error Handling, API / Interface Notes | The PRD notes controlled error responses but it does not define validation patterns for all inputs, bad requests, or malicious payloads. | Attackers can exploit missing input validation and malformed data to trigger unexpected behavior or access issues. This is particularly important for course codes, prerequisite IDs, and AI request payloads. | Medium | Add explicit validation rules for required fields, ID format, duplicate relationships, and request size limits, and document how invalid input is rejected with consistent error codes and safe messages. |
| QS-06 | Error Handling, Failure Scenarios | The PRD describes AI failure and webhook failure in general terms, but does not define retry, timeout, and idempotency behavior for network/database/third-party outages across the full system. | Without clear failure handling, updates may be inconsistent, duplicate webhook deliveries may occur, or users may see broken or partial catalog responses during outages. | High | Specify timeout values, retry policies, idempotency keys, dead-letter or retry logging for failed webhook attempts, and safe fallback behaviors when the database or AI provider is unavailable. |
| QS-07 | Business Rules, Data Integrity | The PRD includes strong integrity rules but it does not yet show how those rules are enforced at the database and service level beyond a few constraints. | A frontend-only validation cannot prevent concurrent writes, duplicate inserts, or race conditions. This is a quality and reliability issue as much as a security issue. | High | State that critical rules such as unique course codes, unique prerequisite pairs, circular dependency detection, and course deletion safety are enforced in the backend and database transaction layer, not only in the UI. |

## Delivery and Document Review

### Summary

The PRD is feasible for a student project and fits the required stack, but the delivery and documentation section still needs clearer operational details. The document explains the product well and stays within a reasonable MVP boundary, yet it should define a realistic development path, release workflow, and recovery approach so that a new team member can understand how the system is deployed and maintained during one semester.

### Findings

| ID | PRD section | Issue or question | Why it matters | Severity | Recommended change |
|---|---|---|---|---|---|
| DD-01 | Deployment, Constraints, Technology Stack | The PRD states a zero-THB deployment budget and proposes Vercel, Supabase, and free-tier services, but it does not describe environment setup, domain configuration, or the minimum viable deployment process in enough detail. | A project can appear feasible on paper but still fail in practice if team members do not know how to bootstrap local and production environments, set secrets, and verify the app. | Medium | Add a short deployment checklist covering required accounts, environment variables, database setup, admin bootstrap, webhook URL configuration, and a simple production launch process. |
| DD-02 | Deployment, Risk Mitigation | The PRD mentions risks and mitigations, but it does not provide a minimum release plan or rollback strategy for the MVP. | A team needs a small, reliable release plan to prevent a broken deployment from blocking the semester schedule or the final demo. | Medium | Include a simple release plan with a staging or local test pass, a defined production deployment step, and a rollback path for broken frontend, API, or database changes. |
| DD-03 | Documentation and Maintainability | The PRD is readable and organized, but it does not identify ownership for backend, frontend, database, AI fallback, and webhook reliability concerns. | In a 3–5 person team, unclear ownership often causes missed decisions and weak maintenance after initial implementation. | Medium | Add a brief ownership section listing who is responsible for API contracts, database integrity, admin auth, webhook monitoring, and release verification. |
| DD-04 | Availability and Failure Handling | The PRD says the system should remain usable under normal workloads but does not state what happens if the AI provider fails, the webhook service fails, or the database becomes temporarily unavailable. | The application may feel incomplete if the team cannot explain what is degraded versus unavailable during operations. | Medium | Add a minimal operational runbook covering degraded modes: public catalog remains available without AI, webhook failures are retried and logged, and the app still works with limited features when optional services are down. |
| DD-05 | Documentation and consistency | The PRD is mostly internally consistent, but some sections still rely on general wording such as “approved public information” or “relevant catalog changes” without a concrete operational definition. | A new team member or reviewer may not know which data is published, what counts as relevant change, or when a webhook must fire. | Low | Define concrete examples for “published,” “approved,” and “relevant catalog changes,” and add a short glossary for important domain terms. |

### Decision requests for the group

- Define a realistic MVP deployment and rollback checklist that fits the semester timeline and the zero-THB constraint.
- Specify who owns release validation, API contracts, and webhook monitoring during implementation.
- Add a short operational guide covering AI fallback, webhook retry behavior, and service degradation.
- Ensure the document reads clearly enough for a new team member to understand the product, system boundaries, and release process without additional verbal explanation.

### Final review status

This PRD is close to ready for implementation, but it should be treated as ready with changes rather than fully final. The strongest remaining work is to make the quality, security, and operational requirements concrete enough to test and execute reliably within the project timeline.

