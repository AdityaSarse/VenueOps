# VenueOps: Product Requirements Document

Author: Aditya Sarse | Version 1.0 | November 2026 | Status: Draft

## 1. Overview

VenueOps is a venue and incident management platform for large venues such as stadiums, campuses and malls. Staff report incidents (fire, medical, security, equipment failure, maintenance), the system alerts the responsible team immediately, tracks the incident through a controlled lifecycle, escalates if nobody responds, and keeps an unalterable audit trail. Managers see operational analytics.

This is a simplified, single-developer version of a real product category. The goal is a working, deployed, well-explained system that demonstrates reliable notification delivery, strict access control, state-machine design and basic production operations.

## 2. Problem statement

On a match day at a stadium with 40,000 fans and 600 staff, incidents are usually passed along by radio, phone calls and chat groups. Nothing is recorded in one place, ownership is unclear, escalation depends on someone remembering, and afterwards nobody can prove what happened or how quickly the team responded.

Example: a steward at Gate 2 sees smoke from a food stall. Today they radio a supervisor, who calls the fire team, and the record is written later from memory. With VenueOps, the steward reports it in the app, the fire team is alerted within seconds, the incident escalates to the shift manager if it is not acknowledged within a set time, and the full timeline is available the next morning.

## 3. Goals and non-goals

### Goals

- G1: Any staff member can report an incident in under 30 seconds.
- G2: The responsible team is notified in real time, and an emergency is never saved without its notification being queued.
- G3: Notifications are never duplicated, even with retries or double submissions.
- G4: Every incident has an owner and a valid status at all times, and only legal status transitions are allowed.
- G5: Every change is recorded in an append-only audit log.
- G6: Access is limited by role and by venue.
- G7: Managers can see response times and incident trends.
- G8: The system is deployed publicly, observable, and has load-test results.

### Non-goals (version 1)

- Native mobile apps or offline mode
- Integrations with cameras, access control or ticketing systems
- Multi-tenant isolation for hundreds of client companies
- Kubernetes, distributed tracing and a full logging stack
- SMS, phone call or email delivery (in-app and push-style real-time only; email is a stretch item)

## 4. Users and roles

| Role | Description | Key permissions |
| --- | --- | --- |
| Reporter (staff) | Steward, cleaner, usher, any employee | Create incidents, view their own incidents, add comments |
| Responder | Member of a security, medical, maintenance or operations team | View incidents assigned to their team, acknowledge, update status, add notes |
| Manager | Venue or shift manager | View all incidents for their venues, assign and reassign, close, cancel, view analytics |
| Admin | System administrator | Manage users, roles, venues, locations, teams, escalation rules, view audit logs |

Rules: a user belongs to one or more venues. Users can only see data for venues they belong to. A reporter cannot close incidents. A responder cannot reassign incidents to other teams.

## 5. Scope

### Must have (MVP)

1. Authentication and role-based access control
2. Venue, location and team management
3. Incident reporting
4. Assignment to a team and manual escalation
5. Enforced status lifecycle
6. Real-time notifications with reliable delivery (outbox pattern)
7. Append-only audit log
8. Search, filtering and pagination
9. Basic analytics dashboard
10. Deployed public instance with a README

### Should have

- Automatic escalation when an incident is not acknowledged in time
- One AI feature: suggest incident type and severity from the free-text description, with a fallback when the model fails
- One load test with before and after numbers

### Could have

- Attachments (photos) on incidents
- Email notifications
- MongoDB for flexible event payloads, with Postgres kept for core data

## 6. User stories

- As a reporter, I can report an incident with type, location, severity and description so the right team responds quickly.
- As a responder, I get an immediate alert when my team is assigned an incident, and I can acknowledge it.
- As a responder, I can move an incident to In Progress and Resolved, and add notes.
- As a manager, I can assign or reassign incidents, and I am alerted when one escalates.
- As a manager, I can close a resolved incident after review.
- As a manager, I can filter incidents by status, type, severity, location and date.
- As a manager, I can see average response time, open incidents and incidents by location and type.
- As an admin, I can create venues, locations, teams and users, and assign roles.
- As an auditor (admin), I can see who changed what and when for any incident, and nobody can edit that history.

## 7. Functional requirements

### 7.1 Authentication and RBAC

- FR-1.1: Users register through admin creation or invite, and log in with email and password.
- FR-1.2: Passwords are hashed with bcrypt or argon2. No plain-text storage.
- FR-1.3: Login returns a short-lived access token (JWT) and a refresh token stored securely.
- FR-1.4: Every API route declares required roles, enforced in middleware.
- FR-1.5: Data access is scoped by venue membership on the server, never only in the UI.
- FR-1.6: Login and report endpoints are rate limited using Redis.

### 7.2 Venue and location management

- FR-2.1: Admins create venues (for example a stadium) with name, address and time zone.
- FR-2.2: A venue has locations in a hierarchy (for example Gate 2, Food Court B, Block C).
- FR-2.3: Admins create teams per venue with a type (security, medical, maintenance, operations).
- FR-2.4: Locations can be deactivated but not deleted if incidents reference them.

### 7.3 Incident reporting

- FR-3.1: A reporter creates an incident with: type, severity (low, medium, high, critical), venue, location, title, description.
- FR-3.2: The system sets reporter, created time and initial status Reported.
- FR-3.3: Each create request carries an idempotency key. Repeating the same key returns the original incident instead of creating a duplicate.
- FR-3.4: A default team is suggested from the incident type and venue routing rules.
- FR-3.5: Reporters and responders can add comments to an incident.

### 7.3.1 Optional AI assistance

- FR-3.6: Given the description, the system suggests type and severity. The reporter can accept or change them.
- FR-3.7: If the model call fails or times out, the form works normally without suggestions.

### 7.4 Assignment and escalation

- FR-4.1: Managers (or the routing rule) assign an incident to a team and optionally to a person.
- FR-4.2: Each incident has an escalation level starting at 0.
- FR-4.3: Each team has an acknowledgement deadline based on severity (for example critical: 2 minutes, high: 5 minutes).
- FR-4.4: If the incident is not acknowledged before the deadline, the escalation job raises the level and notifies the next recipient (team lead, then shift manager).
- FR-4.5: Escalation jobs are idempotent: running one twice does not escalate twice.
- FR-4.6: Managers can escalate manually with a reason.

### 7.5 Status lifecycle

Statuses: Reported, Assigned, In Progress, Resolved, Closed, Cancelled.

| From | To | Who can do it |
| --- | --- | --- |
| Reported | Assigned | Manager, routing rule |
| Reported | Cancelled | Manager (duplicate or false alarm, reason required) |
| Assigned | In Progress | Responder on the assigned team |
| Assigned | Cancelled | Manager (reason required) |
| In Progress | Resolved | Responder on the assigned team (resolution note required) |
| Resolved | Closed | Manager |
| Resolved | In Progress | Manager (reopen, reason required) |

Any other transition is rejected with a clear error. Acknowledgement is recorded as a timestamp (acknowledged\_at) when a responder first accepts an assigned incident, and it stops escalation.

### 7.6 Real-time notifications

- FR-6.1: On assignment, escalation and status change, the relevant users receive a real-time notification through Socket.IO.
- FR-6.2: A notification record is written to an outbox table in the same database transaction as the incident change.
- FR-6.3: A worker reads pending outbox rows, publishes them, and marks them delivered.
- FR-6.4: On failure, the worker retries with exponential backoff up to a maximum, then marks the notification failed and raises an alert.
- FR-6.5: Each notification has a unique key, so duplicates are ignored by the consumer.
- FR-6.6: Users who are offline see unread notifications when they reconnect.

### 7.7 Audit logs

- FR-7.1: Every create, assignment, status change, escalation, comment and role change writes an event row with actor, action, before and after values, and time.
- FR-7.2: Audit rows are append-only. The application database role has no update or delete permission on the table.
- FR-7.3: Admins can view and filter the audit log by incident, user and date range.

### 7.8 Analytics dashboard

- FR-8.1: Open incidents by status and severity.
- FR-8.2: Incidents by type and by location over a chosen period.
- FR-8.3: Average and 90th percentile time to acknowledge and time to resolve.
- FR-8.4: Escalation count and rate.
- FR-8.5: Results are cached in Redis for a short time and invalidated or expired predictably.

### 7.9 Search, filtering and pagination

- FR-9.1: Filter incidents by status, type, severity, team, location, reporter and date range.
- FR-9.2: Text search over title and description (Postgres full-text search).
- FR-9.3: Cursor-based pagination for incident lists, with a stable sort by created time.
- FR-9.4: Indexes support the most common filters.

## 8. Data model

| Table | Key columns |
| --- | --- |
| users | id, name, email, password\_hash, role, is\_active, created\_at |
| venues | id, name, address, timezone |
| user\_venues | user\_id, venue\_id |
| locations | id, venue\_id, parent\_id, name, is\_active |
| teams | id, venue\_id, name, type |
| team\_members | team\_id, user\_id, is\_lead |
| incidents | id, venue\_id, location\_id, type, severity, status, title, description, reporter\_id, assigned\_team\_id, assigned\_user\_id, escalation\_level, acknowledged\_at, resolved\_at, closed\_at, created\_at, updated\_at |
| incident\_comments | id, incident\_id, author\_id, body, created\_at |
| incident\_events | id, incident\_id, actor\_id, action, old\_value, new\_value, created\_at (append-only audit log) |
| outbox | id, event\_type, payload, dedupe\_key, status, attempts, next\_attempt\_at, created\_at |
| notifications | id, user\_id, incident\_id, message, dedupe\_key, read\_at, created\_at |
| escalation\_rules | id, team\_id, severity, ack\_deadline\_seconds, next\_recipient\_type |
| idempotency\_keys | key, user\_id, response\_ref, created\_at |

Important indexes: incidents(venue\_id, status, created\_at), incidents(assigned\_team\_id, status), incidents(location\_id, created\_at), outbox(status, next\_attempt\_at), incident\_events(incident\_id, created\_at), notifications(user\_id, read\_at), and a full-text index on incident title and description.

## 9. API endpoints (version 1)

| Method | Path | Purpose |
| --- | --- | --- |
| POST | /auth/login | Log in |
| POST | /auth/refresh | Refresh access token |
| POST | /auth/logout | Log out |
| GET, POST | /venues | List and create venues |
| GET, POST | /venues/:id/locations | List and create locations |
| GET, POST | /venues/:id/teams | List and create teams |
| POST | /users | Create user (admin) |
| POST | /incidents | Report incident (requires Idempotency-Key header) |
| GET | /incidents | List with filters, search and cursor pagination |
| GET | /incidents/:id | Incident detail with comments and timeline |
| POST | /incidents/:id/assign | Assign to team or user |
| POST | /incidents/:id/acknowledge | Acknowledge assignment |
| POST | /incidents/:id/status | Change status (validated transition) |
| POST | /incidents/:id/escalate | Manual escalation |
| POST | /incidents/:id/comments | Add comment |
| GET | /incidents/:id/events | Audit trail for one incident |
| GET | /audit-logs | Filterable audit log (admin) |
| GET | /notifications | List my notifications |
| POST | /notifications/:id/read | Mark as read |
| GET | /analytics/summary | Dashboard metrics |
| GET | /health | Health check |
| GET | /metrics | Prometheus metrics |

Socket.IO events: incident.assigned, incident.escalated, incident.status\_changed, notification.new.

## 10. Notification and escalation flow

1. The reporter submits the form with an idempotency key.
2. The API validates the request, checks the key, and starts a database transaction.
3. Inside the transaction it inserts the incident, an audit event, and an outbox row for the assignment notification.
4. The transaction commits, and the API returns the incident immediately.
5. A worker polls the outbox (or is triggered by a queue job), creates the notification row, and publishes to the user's Socket.IO room.
6. If publishing fails, the worker increments attempts, schedules the next attempt with backoff, and logs the failure.
7. At the same time, an escalation job is scheduled for the acknowledgement deadline (BullMQ delayed job on Redis).
8. If the incident has been acknowledged when the job runs, the job exits. If not, it raises the escalation level, writes an audit event and outbox row in one transaction, and schedules the next level.
9. Duplicate protection comes from the idempotency key on create, the dedupe key on outbox and notifications, and the check inside the escalation job.

Failure scenarios to handle and test: the API crashes after commit but before publishing (outbox recovers it), Redis is down (jobs retry, API still saves incidents), the worker crashes mid-job (job is retried safely), the same request is sent twice, and Socket.IO clients reconnect after a network drop.

## 11. Non-functional requirements

| Area | Requirement |
| --- | --- |
| Performance | Incident list and create endpoints respond within 300 ms at the 95th percentile under the target load |
| Scale target | 10,000 registered users, 500 concurrent connected users, 50 incident creates per second burst |
| Availability | Single-instance deployment for version 1, with restart on failure and documented recovery steps |
| Reliability | No lost notifications after a committed incident, at-least-once delivery with duplicate suppression |
| Security | Hashed passwords, JWT with expiry, role and venue checks on the server, input validation, rate limiting, CORS and security headers, secrets in environment variables |
| Auditability | Append-only audit log, retained for the life of the project |
| Observability | Structured JSON logs, request IDs, health endpoint, key metrics exposed |
| Maintainability | Modular backend, JavaScript on the frontend, automated tests on core logic |

## 12. Architecture and tech stack

| Layer | Choice | Reason |
| --- | --- | --- |
| Frontend | React + JavaScript | Matches target stack, clear API contracts |
| Backend | Node.js + Express (modular: routes, controllers, services, repositories) | Familiar, clear separation of concerns |
| Database | PostgreSQL | Transactions, constraints, full-text search, indexes |
| Cache and queue | Redis, BullMQ | Caching, rate limiting, delayed escalation jobs |
| Realtime | Socket.IO | Rooms per user and per team |
| Reverse proxy | Nginx | TLS, static files, WebSocket proxying |
| Containers | Docker and Docker Compose | Reproducible local and server setup |
| CI/CD | GitHub Actions | Test, build image, deploy |
| Hosting | Azure Virtual Machine (single instance) | Simple to operate, same Docker Compose setup as local |
| Monitoring | Prometheus and Grafana (basic) | Request rate, latency, queue depth |

Components: web client, API service, worker service, Postgres, Redis, Nginx. The API and worker run from the same codebase as separate processes.

## 13. Security considerations

- Server-side authorization on every route, with automated tests that try forbidden actions.
- Parameterized queries only.
- Request validation with a schema library.
- Rate limiting on login and incident creation.
- No sensitive data in logs. Passwords and tokens are never logged.
- Venue scoping applied in the repository layer so it cannot be forgotten in a controller.
- Dependencies pinned and scanned in CI.

## 14. Testing plan

- Unit tests: status transition rules, permission checks, escalation logic.
- Integration tests: create incident with outbox in one transaction, duplicate create with the same key, forbidden role actions.
- Failure tests: stop Redis, kill the worker, replay a job, send concurrent requests.
- Load test (k6): incident create and list at increasing concurrency. Record p95 latency and error rate, fix one bottleneck (for example a missing index or an uncached analytics query), and record the improvement.
- Manual end-to-end check of the stadium scenario in section 2.

## 15. Success metrics

| Metric | Target |
| --- | --- |
| Report-to-alert latency | Under 2 seconds in normal conditions |
| Duplicate notifications | Zero in duplicate-request tests |
| Invalid status transitions accepted | Zero |
| Unauthorized access in permission tests | Zero |
| p95 latency on list and create | Under 300 ms at target load |
| Load test improvement | At least one documented before and after result |
| Deployment | Public URL working with HTTPS |

## 16. Timeline (11 days, November 7 to 17)

| Day | Focus | Done when |
| --- | --- | --- |
| 1 | Design, schema, API list, state machine diagram, repo, Docker Compose | Diagram and schema exist, containers run |
| 2 | Auth, RBAC, venues, locations, teams | Roles enforced on every route |
| 3 | Incident CRUD, lifecycle, audit log | Invalid transitions rejected, every change logged |
| 4 | Assignment, escalation, outbox and worker | Duplicate requests create no duplicate notifications |
| 5 | Socket.IO, search, filters, pagination | Assigned team receives a live alert |
| 6 | Frontend: login, list, report form, detail | Full flow works in the browser |
| 7 | Analytics dashboard | Counts and response times shown |
| 8 | AI suggestion feature with fallback | Works, and the app still works when the model fails |
| 9 | Deploy to an Azure VM with Nginx and basic CI | Public link works |
| 10 | k6 load test, fix one bottleneck, README | Before and after numbers recorded |
| 11 | Mock interview, walkthrough rehearsal, buffer | Can explain every part without notes |

Cut order if behind: k6 load test, then GitHub Actions, then dashboard polish. Do not cut the state machine, RBAC, outbox flow, audit log or deployment.

## 17. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Scope too large for the time | Build the must-have list first, treat everything else as optional |
| Festival days reduce working time | Shift work forward instead of rushing, keep a daily done-when check |
| Features built without understanding | Write a short design note per module and be able to explain every line |
| Deployment problems late in the schedule | Deploy a skeleton on day 5 or earlier, then redeploy as features land |
| AI feature unreliable | Treat it as optional with a fallback, drop it if time is short |

## 18. Open questions

- Single venue per user or multiple venues per user in the demo data?
- Should auto-escalation notify one level up or all levels above?
- Is email delivery worth adding, or are in-app notifications enough for the demo?
- Which one improvement will the load test target?

## 19. Deliverables

- Working application at a public URL
- Source repository with README, architecture diagram and a decisions and trade-offs section
- ER diagram and status lifecycle diagram
- Load test script and recorded results
- Short incident log of problems found and how they were fixed
- A 3-minute spoken walkthrough prepared for interviews
