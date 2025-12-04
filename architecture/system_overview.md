# System Architecture Document (MVP v1.0)

## 1. System Overview

The architecture is a classic **3-Tier microservice pattern** (simplified to a monolithic backend for MVP) focusing on a secure mobile-first experience.

### System Overview Diagram (Described in Text)

1. **Mobile Frontend App (iOS/Android determined later):** The native application that the user interacts with. It handles the UI, daily check-in workflow, data visualization, and receiving push notifications. It only communicates with the **Backend API** using secure **HTTPS** and **JSON** payloads.
2. **Backend API (Monolith):** The central server-side application. It acts as the business logic layer, handling all requests from the Frontend, performing data validation, calculations (e.g., current week), and coordinating data persistence.
3. **Authentication Service:** A dedicated, external or integrated service (e.g., Firebase Auth, Auth0, or an internal module) used exclusively for **Secure User Registration** and maintaining user sessions. It issues access tokens (e.g., JWT) to the Frontend App.
4. **Database (e.g., PostgreSQL/NoSQL):** The data persistence layer. It securely stores all core entities: User details, Pregnancy Profile, Daily Logs (Symptoms, Mood, Weight, Journal), and Static Content.

## 2. API Endpoint List (REST Format)

These endpoints facilitate the core features (A, B, C, D) of the MVP. All requests require a valid **Authentication Token** in the header.

| Feature Category | Endpoint | Method | Description |
|:---|:---|:---|:---|
| **User & Auth** (A) | `/v1/auth/register` | POST | Secure User Registration (Passes to Auth Service). |
| | `/v1/users/profile` | POST | Create/Update **Pregnancy Profile Setup** (EDD/LMP/Initial Weight). |
| | `/v1/users/profile` | PUT | **Profile Editing** (Update EDD/Weight). |
| | `/v1/users/profile` | GET | Fetch user profile and calculated status (Week X + Y days). |
| **Daily Logging** (B) | `/v1/logs/daily` | POST | **Daily Check-in** - Submit Symptom, Weight, Mood, and Journal entries. |
| | `/v1/logs/weight` | GET | Fetch **Weight Trend Chart** data (last 7/30 days). |
| | `/v1/logs/mood` | GET | Fetch **Mood History Visualization** data (color-coded). |
| | `/v1/logs/summary/pdf` | GET | Generate and return **PDF Data Summary Export**. |
| **Content** (C) | `/v1/content/week/{week_number}` | GET | Retrieve **Week-Specific Content** (Weeks 1-12 only). |
| | `/v1/content/symptom/check` | GET | Get **Symptom Normalcy Guide** based on current week and logged symptom. |
| **Feedback** (D) | `/v1/feedback` | POST | Submit **User Satisfaction Feedback** (NPS/Rating). |

## 3. Entity Descriptions (Data Model Sketch)

The data model is minimal, focusing only on the required fields to meet Acceptance Criteria.

### User (Managed by Auth Service & Backend)

* **user_id** (PK): Unique identifier.
* **email**: User's secure login email.
* **platform_token**: Token for push notifications (**Daily Check-in Reminder**).
* **created_at**: Timestamp.

### PregnancyProfile (One-to-One with User)

* **profile_id** (PK): Unique identifier.
* **user_id** (FK): Link to User.
* **edd**: Estimated Due Date (Date).
* **lmp_start_date**: Last Menstrual Period start date (Date - used for calculation if EDD is missing).
* **initial_weight**: Weight recorded at setup.

### SymptomLog

* **log_id** (PK): Unique identifier.
* **user_id** (FK): Link to User.
* **log_date**: Date of the entry.
* **symptom_type**: Enum/String (e.g., 'Nausea', 'Fatigue').
* **severity_rating**: Integer (e.g., 1-5, as per AC 5.1).

### DailyMetricLog (Captures Mood, Weight, Journal)

* **metric_id** (PK): Unique identifier.
* **user_id** (FK): Link to User.
* **log_date**: Date of the entry.
* **weight_kg**: Float (stores in a common unit, supporting LBS conversion on FE/API).
* **mood**: Enum/String (e.g., 'Happy', 'Anxious', as per AC 7.1).
* **journal_entry**: Text (optional **Free-Form Journaling**).

### WeeklyContent (Static, Curated Content)

* **content_id** (PK): Unique identifier.
* **week_number**: Integer (1-12 only, as per Out-of-Scope).
* **title**: Title of the content (e.g., "Week 8: What to expect").
* **body**: Detailed content text.

## 4. Backend Workflow: User Registration and Profile Setup

This workflow addresses the sequence of actions for **Feature 1 (Secure User Registration)** and **Feature 2 (Pregnancy Profile Setup)**.

1. **Frontend App** sends `POST /v1/auth/register` with `email` and `password`.
2. **Backend API** validates the input and forwards the registration request to the **Authentication Service**.
3. **Authentication Service** creates the user record, securely hashes the password, and returns a unique **user_id** and an **Access Token** to the Backend API.
4. **Backend API** creates the initial record in the **User** table using the new **user_id**.
5. **Frontend App** receives the **Access Token** and immediately sends `POST /v1/users/profile` with `user_id`, `edd` or `lmp_start_date`, and `initial_weight`.
6. **Backend API** receives the request, validates the **Access Token**, and calculates the **current week of pregnancy** based on the provided date (AC 2.1).
7. **Backend API** saves the record into the **PregnancyProfile** table.
8. **Backend API** responds with a success message and the user's initial calculated **Home Screen Status** (current week).

## 5. Frontend Communication Flow: Fetching Home Screen Status and Content

This workflow addresses the sequence for the main dashboard features (**Home Screen Status** and **Week-Specific Content Access**).

1. **Frontend App** launches and verifies the validity of its stored **Access Token**.
2. **Frontend App** sends a request to the **Backend API** to get the user's current status: `GET /v1/users/profile`.
   * *Expected Data:* User's name, **calculated current week (e.g., Week X + Y days)**, and **EDD countdown** (AC 11.1).
3. **Frontend App** displays the status on the dashboard (**Home Screen Status**).
4. Using the calculated current week (e.g., `8`), the **Frontend App** immediately sends a second request to retrieve the relevant content: `GET /v1/content/week/8`.
   * *Backend Logic:* The Backend API ensures the requested week (`8`) does not exceed the user's calculated current week (AC 9.1) before retrieving the corresponding entry from the **WeeklyContent** table.
   * *Expected Data:* Title and Body text for Week 8 content.
5. **Frontend App** displays the **Week-Specific Content** on the Home Screen or a linked view.