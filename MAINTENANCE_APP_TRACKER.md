# Maintenance Frappe App — Progress Tracker

Started: 2026-11-22
Bench: /Users/mahmoudgaber/frappe/frappe-bench
Site: maintenance.localhost (apps: frappe, maintenance; developer_mode on)
Spec: ~/Downloads/مشروع تدريبي نظام طلبات الصيانة (Frappe + Flutter).pdf
Scope: **backend only** (Flutter app, phase 4, is out of scope)

## Phases (each is a separate PR in the spec)

- [x] Setup: app `maintenance` created, installed on maintenance.localhost
- [x] Phase 1 — DocTypes & permissions
- [ ] Phase 2 — Server logic & Workflow
- [ ] Phase 3 — API (endpoints + Postman collection)

- [—] Phase 4 — Flutter app (SKIPPED)

- [ ] Phase 5 — Background jobs & reports
- [ ] Phase 6 — Deployment (nginx + HTTPS, DEPLOY.md)
- [ ] Phase 7 — SOLID refactor & tests

## Spec summary

**Roles:** Maintenance Customer (own requests only), Maintenance Supervisor (all requests, Desk; assigns, sets priority, closes), Maintenance Technician (only assigned tasks), System Manager. Nobody sees others' data, via Desk or API.

**Statuses:** Open → Assigned → In Progress → Completed → Closed; Supervisor can Cancel from any status before Completed. Any other transition is rejected.
Transition owners: Assign/Close/Cancel = Supervisor only; Start/Complete = assigned Technician only.

**DocTypes:**

- Maintenance Customer: customer_name, mobile (unique), address, user (Link User)
- Technician: user (Link User, unique), full_name, specialty (Select: AC / Plumbing / Electrical), is_available (Check)
- Maintenance Request (naming MR-.YYYY.-.####): customer, title, description, priority (Low/Medium/High/Urgent), status, technician, due_by, before_photo, after_photo, parts (table), total_cost (read-only)
- Request Part (child): part_name, qty, rate, amount (computed)
- Request Status Log: request, from_status, to_status, changed_by, changed_on, note

**Server rules (apply from Desk and API alike):**

- amount = qty × rate; total_cost = sum of parts (automatic)
- Cannot assign without a technician, nor to an unavailable technician
- Cannot complete without an after photo
- due_by auto-set on create: Urgent +4h, High +24h, Medium +48h, Low +72h
- Every status change logged in Request Status Log (who, when, from, to, note)
- In-system notification to technician on assignment
- Desk: technician picker shows only available + matching specialty; status colors in list view

**API** (token auth, not cookies; paths `maintenance.api.<module>.<fn>`):
auth.login, tasks.my_tasks (page/page_size/status), tasks.get_task, tasks.update_status, tasks.upload_photo (before/after), tasks.add_parts, requests.create, requests.my_requests, dashboard.stats.
Response shape always `{ok, data}` or `{ok:false, error:{code,message}}` with proper HTTP status (400 bad data, 403 permission, 404 missing). Thin API layer (validate → call logic), never bypass permissions, paginated lists, Postman collection for all endpoints.

**Background jobs & reports (Phase 5):** on Completed, enqueue PDF report email to customer (never block the request); hourly SLA check notifies supervisor once per overdue request; Script Report "Technician Performance" (completed count, avg hours, SLA %; filters: period, specialty); Print Format for repair report.

**Deployment (Phase 6):** production setup behind nginx, HTTPS with http→https redirect, photo upload works, survives reboot (supervisor/systemd), DEPLOY.md explaining each nginx part, logs, restarts, cert renewal.

**Deliverables:** Git repo, README (local run), Postman collection, DEPLOY.md, tests.

## Decisions

- Own DocTypes (no ERPNext dependency); optional extra: link Customer to ERPNext.

## Log

- 2026-11-22: Tracker created.
- 2026-11-22: App `maintenance` and site `maintenance.localhost` created, installed, verified.
- 2026-11-22: Read project spec PDF; scope set to backend only (skip Flutter phase 4).
- 2026-11-22: Phase 1 started: user created DocType(s) in Desk (Link field needs Options = target DocType). Now learning Workflow (Phase 2).
- 2026-11-23: Phase 3 started: api/utils.py (ok/fail) and api/auth.py login written up to credential + enabled checks (key/secret/roles still to do).
- 2026-11-23: auth.login complete (key/secret/roles returned); awaiting curl tests.
- 2026-11-23: auth.login tested OK for customer1/tech1/supervisor1 (200, roles correct, 400 missing, 401 wrong pwd, token works). Linked Maintenance Customer/Technician records still to create. Next: requests.create.
- 2026-11-24: tasks.my_tasks part 1 done (technician lookup, tested). Decision: page_size fixed server-side at 10 (PAGE_SIZE constant), client sends only page.
- 2026-11-24: tasks.my_tasks part 2 done (status/page validation, tested: 200 valid, 400 INVALID_STATUS / INVALID_PAGE). Next: part 3, query DB filtered by technician+status, paginated, return list + total.
- 2026-11-24: tasks.my_tasks part 3 written (get_list filtered by technician/status, paginated 10, total). Decision: no default status (all statuses when none sent). Needs test data to verify non-empty results.
- 2026-11-24: tasks.py refactored into helpers (_get_technician, _validate_status, _validate_page, _fetch_tasks); my_tasks only orchestrates. Regression tested.
- 2026-11-24: auth.py refactored into helpers (_validate_fields/_credentials/_enabled, _generate_api_keys, _get_roles); login only orchestrates. Regression tested. Decision: Postman collection to be built at the end for all endpoints.
- 2026-11-24: tasks.get_task written (helpers: _validate_name, _validate_request_exists, _validate_ownership, _serialize_task). Error cases tested; success + ownership cases need test data.
- 2026-11-24: Decision: Phase 2 controller rules first (option 1), then tasks.update_status. Controller step 1 done: calculate_totals (amount=qty*rate, total_cost=sum) in maintenance_request.py. Next: due_by, assignment rules, completion rule, assigned-technician-only, status log.
- 2026-11-24: controller step 2 done: before_insert -> set_due_by (Urgent 4h/High 24/Medium 48/Low 72, default Medium). User preference: use small edits, don't rewrite whole files; no git commits by me.
