# FitIntelligence — Technical Documentation

FitIntelligence is a smart gym split calendar, membership tracker, and automated reminder application featuring GPS geofencing and Gemini AI-powered selfie verification.

---

##  System Architecture

The project is structured as a decoupled client-server application optimized for reliability, real-time background scheduling, and automated containerized deployment.
<img width="1536" height="1024" alt="fitellinengencearchitect" src="https://github.com/user-attachments/assets/941bd609-d195-406d-a837-d1535564b5b3" />


### Key Technical Choices
* **Frontend**: Bare React Native with TypeScript, using Expo libraries (e.g. for push notifications, location) to access native APIs while preserving full control over the Android/iOS directories. Zustand is used for clean, lightweight state management.
* **Backend**: FastAPI (Python 3.11+) for high-performance, asynchronous REST API endpoints, utilizing Pydantic models for request/response validation.
* **Database & ORM**: PostgreSQL database mapped via SQLAlchemy asynchronous ORM. Database migrations are managed via Alembic.
* **Task Scheduling**: Redis acts as a message broker for Celery. Celery Beat executes periodic tasks (like gym membership reminders and alarm checks) while Celery Workers handle async message dispatching.
* **AI Analysis**: Gemini 1.5 Flash is integrated directly for rapid, multimodal analysis of gym selfie uploads to verify workouts.

---

##  Database Schema & Models

Below is the entity-relationship model of the Postgres database.

```mermaid
erDiagram
    users ||--o| gym_info : "has one"
    users ||--o{ workout_splits : "owns"
    users ||--o{ reminders : "owns"
    users ||--o{ login_logs : "generates"
    workout_splits ||--o{ split_days : "contains"

    users {
        uuid id PK
        string email
        string full_name
        string hashed_password
        boolean is_active
        boolean is_admin
        datetime created_at
    }

    gym_info {
        uuid id PK
        uuid user_id FK
        string name
        string address
        float latitude
        float longitude
        date start_date
        date end_date
    }

    workout_splits {
        uuid id PK
        uuid user_id FK
        string name
        boolean is_active
        datetime created_at
    }

    split_days {
        uuid id PK
        uuid split_id FK
        integer day_index
        string workout_type
        string description
    }

    reminders {
        uuid id PK
        uuid user_id FK
        time trigger_time
        boolean is_strict
        boolean is_dismissed
        datetime verified_at
    }

    login_logs {
        uuid id PK
        uuid user_id FK
        datetime logged_at
        float latitude
        float longitude
        string resolved_address
    }
    
    registered_gyms {
        uuid id PK
        string name
        string address
        float latitude
        float longitude
        json packages
        json amenities
    }
```

---

##  Core Feature Workings

### 1. Gym Subscription Tracker
* Users input their gym's details, registration dates, package duration, and coordinates.
* The frontend calculates days remaining on the package. Card colors dynamically shift to warn users:
  * **Green**: > 7 days remaining
  * **Yellow**: 1 to 7 days remaining
  * **Red**: 0 days or expired

### 2. Split Builder & Calendar
* Five preset workout splits are provided (Push-Pull-Legs, Bro Split, Upper-Lower, Full Body, Arnold Split).
* Users can build custom splits mapping daily workout types (e.g. Chest/Triceps, Rest Day) to indexes from Monday to Sunday.
* The mobile console renders a 7-day horizontal calendar, highlights active workout days, and lists corresponding focus areas.

### 3. Smart Reminders & Verification
* **Standard Mode**: Reminders send standard dismissable push notifications.
* **Strict Mode**: Activates a full-screen lock overlay blocking application usage. To unlock the app, users must verify their gym presence using one of two methods:
  * **GPS verification**: Calculates the user's distance using the Haversine formula. The user must be within 300 meters of the registered gym's coordinates.
  * **Gemini AI Selfie Verification**: The user takes a gym workout selfie. The app uploads it to the backend where Gemini 1.5 Flash analyzes the image. Gemini responds with a JSON object (`{"is_gym": true/false, "explanation": "..."}`) to approve or reject the verification.

### 4. Admin Console & Analytics
* Restricted strictly to administrative users (`is_admin=True`).
* Displays registration analytics based on local system calendar boundaries (resetting exactly at midnight local time):
  * **New Today**: Registered since midnight local time today.
  * **New This Week**: Registered since Sunday midnight local time of the current week.
  * **New This Month**: Registered since 1st day of the current month.
  * **New This Year**: Registered since Jan 1st of the current year.
* Renders **Active Users & Geolocation Logs** showing full user lists, registration details, last active times, and map buttons to view user coordinates in Google or Apple Maps.

---

##  Production Deployment (AWS EC2 & ECR)

The application is deployed on an **AWS EC2** instance via an automated **GitHub Actions** CI/CD pipeline.

```mermaid
sequenceDiagram
    Participant Dev as Developer
    Participant Git as GitHub Actions
    Participant ECR as AWS ECR
    participant EC2 as AWS EC2

    Dev->>Git: git push origin main
    activate Git
    Git->>Git: Run Linters & Pytest Suite
    Git->>Git: Build Production Docker Image
    Git->>ECR: Push Docker Image (with commit SHA tag)
    
    Note over Git,EC2: Continuous Delivery Starts
    Git->>EC2: Sync code files via Rsync (SSH)
    Git->>EC2: Log in to ECR & Pull new image
    Git->>EC2: docker compose up -d (Rolling update)
    Git->>EC2: Run Alembic migrations
    
    Git->>EC2: Poll Health Check Endpoint (/health)
    alt Health Check Fails
        Git->>EC2: Revert git HEAD & Restart previous container
        Git-->>Dev: Alert Deployment Failure & Rollback
    else Health Check Success
        Git-->>Dev: Deployment Successful!
    end
    deactivate Git
```

### Deployment Configuration Details
* **Multi-Container Composition**: Configured via `docker-compose.yml` specifying isolated networks, volume persistence for Postgres and Redis, and startup dependencies (healthcheck blocks).
* **Rollback Policy**: If the HTTP healthcheck fails 10 times consecutively, the workflow connects back to EC2, stashes changes, checkouts the previous git commit, and rebuilds/restarts containers to ensure zero downtime.
* **Environment Configuration**: Sensitive secrets (JWT secret, DB password, Gemini Key) are injected via GitHub Repository Secrets and compiled dynamically.

---

##  Local Setup and Administration

### Bootstrapping Administrative Accounts
To grant admin privileges to an account, use the utility script provided in `backend/promote_admin.py`.

```bash
cd backend
python promote_admin.py --email user@example.com
```
*Note: If the email does not exist in the database, the script will automatically register them and output a default password (`Admin123!`).*
