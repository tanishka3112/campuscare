# CampusCare

**An AI-enabled appointment and queue management system for a university hospital and dispensary.**

Students, faculty, staff and their authorised dependants can find a doctor, book a slot, and track their place in the queue. Doctors, reception staff and administrators get the tools they need to run the day. A booking can never be double-booked, even when many people try for the same slot at once.

> **Status: in development** (started November 2026). Nothing below is marked done until it is built and tested. Check the [roadmap](#roadmap) for progress.

Built by Tanishka Menon as an MCA major project at Indira Gandhi Delhi Technical University for Women.

<!-- Add a demo GIF here once the first slice works: ![demo](docs/demo.gif) -->

---

## Table of contents

- [Why this project](#why-this-project)
- [Features](#features)
- [Architecture](#architecture)
- [Data model](#data-model)
- [Tech stack](#tech-stack)
- [Getting started](#getting-started)
- [Project structure](#project-structure)
- [Running the tests](#running-the-tests)
- [AI components](#ai-components)
- [Engineering decisions](#engineering-decisions)
- [Scope and limitations](#scope-and-limitations)
- [Roadmap](#roadmap)

---

## Why this project

Campus clinics often run on phone calls and paper queues. That causes double-bookings, long unexplained waits, and no clear picture of how busy each doctor is. CampusCare focuses on three things:

1. **Reliable booking.** One slot, one patient, enforced by the database and not only by application code.
2. **Visible queues.** Patients can see where they stand, and reception can see the whole day.
3. **AI that is measured, not assumed.** Every AI feature is compared against a simple baseline and reported honestly, including where it fails.

---

## Features

Status key: `[ ]` planned, `[x]` built and tested.

**Booking**
- [ ] Search doctors by specialisation, facility, gender, language, and date
- [ ] View a doctor's open slots
- [ ] Book and cancel appointments
- [ ] Concurrency-safe booking (no double-booking under simultaneous requests)
- [ ] Appointment history and status tracking

**Roles**
- [ ] Registration and login (JWT)
- [ ] Role-based access: patient, doctor, reception, admin
- [ ] Dependants linked to an authorising staff or faculty member

**Clinic operations**
- [ ] Check-in and queue management with a live position for each patient
- [ ] Doctor dashboard for today's appointments and status updates
- [ ] Admin tools for doctors, facilities, weekly schedules, and reports
- [ ] Audit log of who viewed or changed what
- [ ] Booking confirmations

**AI (kept separate from booking)**
- [ ] RAG assistant for university healthcare policies, with citations
- [ ] ML waiting-time prediction
- [ ] Agent that proposes bookings for a patient to confirm (stretch)
- [ ] Offline RL scheduling simulator (stretch)

**Engineering**
- [ ] Automated tests with coverage reporting
- [ ] Docker Compose setup
- [ ] CI with GitHub Actions
- [ ] Live deployment

---

## Architecture

```
  React app (runs in the browser)
            |  JSON over HTTP
            v
   FastAPI backend
   |-- Auth and roles
   |-- Booking, queue, admin
   |-- AI module (separate)  -->  RAG, wait-time model, agent
            |
            v
       PostgreSQL
```

The AI module sits beside the booking code and talks to it through the same API as any other client. If the AI module is slow or down, bookings keep working.

---

## Data model

Nine tables. The first six form the core slice, and the rest are added as the matching features are built.

```mermaid
erDiagram
  users ||--o| doctors : "is a"
  specialisations ||--o{ doctors : has
  doctors ||--o{ doctor_schedules : has
  facilities ||--o{ doctor_schedules : "runs at"
  doctors ||--o{ slots : offers
  facilities ||--o{ slots : hosts
  slots ||--o| appointments : "booked as"
  users ||--o{ appointments : books
  users ||--o{ audit_logs : performs
  appointments ||--o{ notifications : triggers
  users ||--o{ users : sponsors

  users {
    int id PK
    string name
    string email UK
    string password_hash
    string role
    string patient_type
    int sponsor_id FK
  }
  doctors {
    int id PK
    int user_id FK, UK
    int specialisation_id FK
    string gender
    string languages
  }
  specialisations {
    int id PK
    string name
  }
  facilities {
    int id PK
    string name
    string opening_hours
  }
  doctor_schedules {
    int id PK
    int doctor_id FK
    int facility_id FK
    int weekday
    time start_time
    time end_time
    int slot_minutes
  }
  slots {
    int id PK
    int doctor_id FK
    int facility_id FK
    datetime start_time
    datetime end_time
  }
  appointments {
    int id PK
    int slot_id FK
    int patient_id FK
    string status
    datetime created_at
    datetime checked_in_at
    datetime started_at
    datetime completed_at
  }
  audit_logs {
    int id PK
    int user_id FK
    string action
    string entity
    int entity_id
    datetime created_at
  }
  notifications {
    int id PK
    int user_id FK
    int appointment_id FK
    string message
    datetime sent_at
  }
```

**Rules enforced in the database**

```sql
-- One active booking per slot. Cancelled appointments don't count.
CREATE UNIQUE INDEX one_active_booking
ON appointments (slot_id) WHERE status <> 'Cancelled';

-- A doctor can't have two slots starting at the same time.
ALTER TABLE slots ADD CONSTRAINT one_slot_per_doctor_time
UNIQUE (doctor_id, start_time);
```

**Appointment statuses:** Booked, Checked-in, In Progress, Completed, Cancelled, No-show. Only valid transitions are accepted.

---

## Tech stack

| Area | Tools |
|---|---|
| Frontend | React, Vite, React Router |
| Backend | Python, FastAPI, Pydantic |
| Database | PostgreSQL, SQLAlchemy, Alembic |
| AI | Embeddings, ChromaDB, scikit-learn |
| Testing | pytest, pytest-cov |
| Delivery | Docker, Docker Compose, GitHub Actions |

---

## Getting started

**Prerequisites:** Python 3.12, Node.js 20 or later, PostgreSQL 15 or later, and Git.

**Backend**

```bash
git clone https://github.com/<your-username>/campuscare.git
cd campuscare/backend

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # then edit the values
createdb campuscare              # or create it in pgAdmin
alembic upgrade head
python -m app.seed               # synthetic demo data
uvicorn app.main:app --reload
```

The API docs open at http://127.0.0.1:8000/docs.

**Frontend** (in a second terminal)

```bash
cd campuscare/frontend
npm install
cp .env.example .env             # VITE_API_URL=http://127.0.0.1:8000
npm run dev
```

The web app opens at http://localhost:5173.

**Example backend `.env`**

```
DATABASE_URL=postgresql+psycopg://postgres:password@localhost:5432/campuscare
SECRET_KEY=change-me
CORS_ORIGINS=http://localhost:5173
```

Never commit your real `.env` files.

---

## Project structure

```
campuscare/
|-- backend/
|   |-- app/
|   |   |-- main.py          # app entry point and /health
|   |   |-- config.py        # settings from .env
|   |   |-- database.py      # engine and session
|   |   |-- models/          # one SQLAlchemy model per table
|   |   |-- schemas/         # Pydantic request and response models
|   |   |-- routers/         # doctors, slots, appointments, auth, admin
|   |   |-- services/        # booking and queue logic
|   |   |-- ai/              # RAG, wait-time model, agent (separate from booking)
|   |   `-- seed.py          # synthetic demo data
|   |-- migrations/          # Alembic migrations
|   |-- tests/
|   |-- requirements.txt
|   `-- .env.example
|-- frontend/
|   |-- src/
|   |   |-- api/             # functions that call the backend
|   |   |-- components/      # reusable pieces
|   |   |-- pages/           # Login, Search, Book, MyAppointments, DoctorDashboard, Queue, Admin
|   |   |-- App.jsx
|   |   `-- main.jsx
|   |-- index.html
|   |-- package.json
|   `-- .env.example
|-- docs/                    # decision log, evaluation reports, diagrams
|-- .github/workflows/       # CI
|-- docker-compose.yml       # added later
`-- README.md
```

---

## Running the tests

```bash
cd backend
pytest --cov=app
```

Tests run against a separate test database. The concurrency test fires 100 simultaneous booking requests at one slot and checks that exactly one succeeds.

---

## AI components

All AI features are optional layers. Results will be filled in as each is built, and they will include cases where the AI fails.

### RAG healthcare information assistant
Answers questions about policies, facility timings, services, and appointment and cancellation rules using approved documents. It cites its sources, says "I don't know" when the documents don't cover a question, and refuses to give diagnosis or treatment advice.

| Metric | Keyword baseline | RAG |
|---|---|---|
| Answer correctness | TBD | TBD |
| Citation correctness | TBD | TBD |
| Correct abstentions | TBD | TBD |
| Correct refusals (unsafe questions) | TBD | TBD |

### Waiting-time prediction
Estimates how long a checked-in patient will wait before consultation. Trained and tested on synthetic data, split by time.

| Model | MAE (minutes) |
|---|---|
| Overall average (baseline) | TBD |
| Average per hour (baseline) | TBD |
| Linear regression | TBD |
| Random forest | TBD |
| Gradient boosting | TBD |

### Agent (stretch)
Helps a patient find and propose a booking. It can search and propose, but it never books. The patient confirms through the normal booking endpoint.

### RL scheduling simulator (stretch)
An offline simulator that compares reinforcement learning against first-available and load-balancing schedulers. It never changes real appointments, and a result where RL doesn't win is still reported.

---

## Engineering decisions

A running log of what was chosen and why lives in [`docs/decisions.md`](docs/decisions.md). Highlights:

- **Booking safety is enforced by PostgreSQL**, using a unique index and row locking, not only by checks in code.
- **A partial unique index** lets a cancelled slot be booked again while still blocking double bookings.
- **AI is separate from booking**, so an AI failure can never corrupt an appointment.
- **Facilities are a table of two rows** (Campus Hospital and Campus Dispensary) so admins can manage hours and the assistant can answer questions about them.

---

## Scope and limitations

- CampusCare is an **appointment and queue system**. It is not a hospital information system or an electronic medical record.
- **Out of scope:** diagnosis, treatment recommendations, emergency dispatch, billing, insurance, and pharmacy inventory.
- **No medical details are stored.**
- **Demo data and model evaluations use synthetic data**, and every result that depends on it is labelled as synthetic.
- AI features are advisory and offline from the booking logic.

---

## Roadmap

- [ ] Foundation: repo, database, models, seed data
- [ ] Doctor search and open slots
- [ ] Booking with no double-booking, and its tests
- [ ] Login and roles
- [ ] Cancellation and the status machine
- [ ] Check-in and queue
- [ ] Doctor, reception, and admin views
- [ ] Web pages
- [ ] Docker, CI, and deployment
- [ ] RAG assistant and evaluation
- [ ] Wait-time model and evaluation
- [ ] Agent (stretch)
- [ ] RL simulator (stretch)
- [ ] Project report and viva preparation

---

## License

Add a license of your choice, for example MIT, before making the repo public.
