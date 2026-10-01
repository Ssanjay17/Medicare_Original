# MediCare+ Frontend

A React (Vite) frontend wired to every endpoint in the MediCare+ FastAPI/MongoDB backend.

## Design

Clinical-records aesthetic: `Newsreader` serif for headers, `IBM Plex Sans` body, `IBM Plex Mono` for
IDs/invoices/timestamps. Deep teal primary, amber accent. The signature element is the **Pathway**
stepper (see `src/components/Pathway.jsx`) — it mirrors the backend's actual appointment state machine
(`booked → checked_in → in_consultation → completed`, with `cancelled` as a branch) on every appointment
detail page.

## Setup

```bash
npm install
cp .env.example .env   # set VITE_API_BASE_URL to your backend URL
npm run dev
```

The backend must be running (see `../medicare-plus/README.md`) with CORS open to the frontend origin —
already configured with `allow_origins=["*"]` in `app/main.py`.

## Roles covered

Every role from the OpenAPI spec has a tailored nav and page set:

- **Patient** — find/book doctors, my appointments, my profile (tabs for profile / medical history /
  lab results / prescriptions), my invoices.
- **Doctor** — appointment queue, patient directory + chart, profile & schedule management, order/review
  lab tests, write consultation notes and prescriptions.
- **Nurse / Receptionist** — appointment queue (check-in, start consultation, cancel), patient directory,
  billing (receptionist: generate invoices, confirm payments).
- **Lab Technician** — lab queue (collect sample → upload result), filterable by status.
- **Pharmacist** — inventory (add medicine, restock/dispense, low-stock filter), prescription lookup.
- **Admin** — everything above, plus the reports dashboard (stat cards, appointments trend line chart,
  department-wise donut chart, recent activity) and user management (activate/deactivate any account).

## Structure

```
src/
  api/            axios client (JWT interceptor) + one function per backend route
  context/        AuthContext (login/register/logout, persisted session)
  components/     Layout shell, Pathway stepper, Modal, StatusBadge, shared CSS
  pages/          one page per feature area, routed by role in App.jsx
```

## Notes

- Two backend endpoints were added to support staff queue views that the original API didn't expose:
  `GET /api/appointments/all` and `GET /api/lab-tests/all` (both role-guarded the same way as their
  sibling endpoints). See `../medicare-plus/app/routers/appointments.py` and `lab_tests.py`.
- This environment has no outbound access to install/run a local MongoDB, so the build was verified via
  `npm run build` (clean) and the backend's own test suite (`pytest`, 7/7 passing) rather than a live
  end-to-end request. Run both servers locally to do a full smoke test.


# MediCare+ — Hospital Management System (API)

FastAPI + MongoDB backend implementing the business flow, architecture, and features
from the MediCare+ diagram. **No containerization yet** — this is the plain
Python/Jenkins-CI stage; Podman/OpenShift comes later when you say so.

## Stack
- **Python 3.12 / FastAPI** — app layer (stands in for the API Gateway + microservices
  in the diagram; currently a single deployable, structured into per-domain routers
  so it can be split into real microservices later without rewriting logic).
- **MongoDB (Motor, async driver)** — the *only* database, per your instruction. The
  diagram's relational ER tables (users/patients/doctors/appointments/prescriptions/payments)
  are modeled as MongoDB collections with string-ID references instead of foreign keys.
- **JWT (python-jose) + bcrypt (passlib)** — auth & role-based access control.
- **Jenkins** — CI pipeline (lint, compile check, unit tests, artifact packaging).

## Business flow implemented (diagram section 1 & 4)
1. **Patient registers/logs in** → `POST /api/auth/register`, `/api/auth/login`
2. **Appointment booking** (choose doctor, date, time) → `POST /api/appointments`
   (with slot clash protection + doctor schedule sync)
3. **Doctor consultation** (checks history, writes notes) →
   `GET /api/patients/{id}` (history), `PATCH /api/appointments/{id}/consultation`
4. **Tests & diagnostics** (if required) → `POST /api/lab-tests`,
   `PATCH /api/lab-tests/{id}/collect-sample`
5. **Results uploaded** → `PATCH /api/lab-tests/{id}/upload-result`,
   `PATCH /api/lab-tests/{id}/review`
6. **Prescription & treatment** → `POST /api/prescriptions`
7. **Billing & payment** (online/offline) → `POST /api/billing/generate`,
   `POST /api/billing/confirm`
8. **Follow-up & reports** → `GET /api/lab-tests/patient/{id}`,
   `GET /api/prescriptions/patient/{id}`, `GET /api/billing/patient/{id}`

Appointment status is a strict state machine (`app/models/appointment.py`):
`booked → checked_in → in_consultation → completed`, with `cancelled` allowed only
from `booked`/`checked_in`. Illegal transitions (e.g. skipping consultation) are rejected.

## Roles (diagram section 1: Patients, Doctors, Nurses, Receptionists, Lab Technicians,
Pharmacists, Admin) are enforced via `app/core/deps.py` guards on every route.

## MongoDB collections (adapted from section 6 ER design)
`users`, `patients`, `doctors`, `appointments`, `lab_tests`, `prescriptions`,
`pharmacy`, `payments`, `notifications`, `counters` (atomic invoice numbers),
`audit_logs`.

## Dashboard / Reports (diagram section 7)
`GET /api/reports/dashboard` — total patients/doctors, appointments today, revenue MTD
`GET /api/reports/appointments-trend` — line chart data
`GET /api/reports/department-wise-patients` — donut chart data
`GET /api/reports/recent-appointments`, `/api/reports/recent-payments`

## Local setup
```bash
cp .env.example .env          # edit MONGO_URI, JWT_SECRET_KEY
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python seed.py                 # demo admin/doctors/patient/pharmacy stock
uvicorn app.main:app --reload  # http://localhost:8000/docs
```

Demo logins (from `seed.py`): `admin@medicareplus.com` / `Admin@123`,
`dr.smith@medicareplus.com` / `Doctor@123`, `john.doe@example.com` / `Patient@123`.

## CI (Jenkins)
`Jenkinsfile` runs: checkout → venv setup → flake8 lint → compile check → pytest
(with JUnit report) → zip artifact archive. A **Build Container** stage exists but
is disabled (`when { expression { return false } }`) — flip that once containers
are wanted, matching diagram section 3 (Podman network: frontend / API gateway /
microservices / Postgres→Mongo / Redis / MinIO containers).

Required Jenkins credentials: `medicare-mongo-uri`, `medicare-jwt-secret`.

## Not yet built (waiting on your go-ahead)
- Podman/Docker containerization (section 3)
- OpenShift deployment manifests (section 8)
- React frontend, payment gateway (Razorpay) live integration, SMS/email gateway wiring
  — notification and payment confirmation endpoints exist and log to Mongo (`notifications`,
  `payments` collections) as integration points, ready to plug in real providers.
