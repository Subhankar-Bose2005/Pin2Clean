# Pin2Clean

> **Turn a photo into a pinned cleanup request — no address needed.**

Pin2Clean is a civic cleanliness reporting platform that lets citizens report dirty public spaces by simply uploading a photo. It pulls GPS coordinates directly from the image's EXIF data, drops a marker on a live map, and routes it to volunteers who can claim and close the report.

No manual pinning. No address typing. Just take a photo and submit.

---

## Why I Built This

Most civic complaint systems are forms — long, confusing, and slow. People give up halfway. I wanted something closer to how people actually behave: take a photo, tap submit, done.

The hard part was making sure location data is accurate and trustworthy. Client-side EXIF reading is fast and gives users immediate feedback, but it can be spoofed. So the backend re-extracts GPS data independently from the uploaded image before writing anything to the database. Two reads, zero trust in the frontend.

---

## What It Does

| Feature | Details |
|---|---|
| 📸 Photo-based reporting | Upload an image → GPS auto-extracted from EXIF |
| 🗺️ Live map | Leaflet.js + OpenStreetMap, red/green markers per status |
| 👥 Role system | Citizen → report, Volunteer → clean, Admin → moderate |
| ☁️ Cloud storage | Images stored in Supabase Storage, publicly accessible |
| 🔐 JWT auth | Supabase tokens validated server-side via Spring Security |
| 🛡️ Row-Level Security | Postgres RLS policies enforce access at DB level |

---

## Tech Stack

**Backend**
- Java 17 + Spring Boot 3.2.5
- Spring Security (OAuth2 Resource Server / JWT)
- JPA with Hibernate
- `metadata-extractor` library for EXIF parsing

**Database / Auth / Storage**
- Supabase (PostgreSQL + Auth + Storage)

**Frontend**
- Vanilla HTML, CSS, JavaScript (no framework — keeps it lightweight)
- Leaflet.js + OpenStreetMap for mapping
- `exifr.js` for client-side GPS preview before upload

---

## How a Report Works (end to end)

```
User logs in via Supabase Auth
     ↓
Uploads a JPEG/PNG with GPS metadata
     ↓
Frontend reads EXIF → shows location preview on map (UX only)
     ↓
Image sent to backend as multipart form
     ↓
Backend re-extracts GPS from EXIF using metadata-extractor
     ↓
Image uploaded to Supabase Storage → public URL generated
     ↓
Report row written to DB: lat, lng, image_url, severity, status=ACTIVE
     ↓
Red marker appears on the shared map
     ↓
Volunteer visits map → clicks marker → marks as CLEANED
     ↓
Marker turns green, cleaned_by field updated
```

---

## Database Schema

### `profiles`

Created automatically via Supabase trigger on user signup.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | FK → auth.users |
| `email` | text | |
| `role` | text | `CITIZEN`, `VOLUNTEER`, `ADMIN` |
| `created_at` | timestamptz | |

Roles are stored in the database — not in JWT claims. This means you can change a user's role without forcing them to re-login.

### `reports`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `image_url` | text | Supabase Storage public URL |
| `latitude` | float8 | Extracted from EXIF |
| `longitude` | float8 | Extracted from EXIF |
| `description` | text | Optional user note |
| `severity` | text | `LOW`, `MEDIUM`, `HIGH` |
| `status` | text | `ACTIVE` or `CLEANED` |
| `reported_by` | uuid | FK → profiles |
| `cleaned_by` | uuid | FK → profiles (nullable) |
| `created_at` | timestamptz | |
| `cleaned_at` | timestamptz | (nullable) |

**RLS policies:**
- Public: `SELECT` allowed for all
- Authenticated users: `INSERT` allowed
- Volunteers + Admins only: `UPDATE` allowed

---

## Project Structure

```
pin2clean/
│
├── frontend/
│   ├── login.html          # Auth entry point
│   ├── index.html          # Main map interface
│   ├── auth.js             # Supabase login/logout/session
│   ├── app.js              # Map rendering, report submission, marker logic
│   └── style.css
│
└── src/main/java/com/example/pin2clean/
    ├── controller/         # REST endpoints
    ├── model/              # JPA entities
    ├── repository/         # Spring Data repositories
    ├── service/            # Business logic (EXIF parsing, storage upload, report creation)
    └── security/           # JWT filter, role resolution
```

The backend strictly follows a controller → service → repository pattern. No database calls in controllers, no business logic in repositories.

---

## Running Locally

### Prerequisites

- Java 17+
- Maven
- A Supabase project (free tier works)
- Supabase Storage bucket named `report-images` (set to public)
- GPS-enabled device (or a test image with EXIF GPS data)

### 1. Configure the backend

In `src/main/resources/application.properties`:

```properties
supabase.url=https://your-project.supabase.co
supabase.key=your-service-role-key
supabase.jwt.secret=your-jwt-secret

spring.datasource.url=jdbc:postgresql://db.your-project.supabase.co:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=your-db-password
```

### 2. Start the backend

```bash
mvn spring-boot:run
```

Backend runs on `http://localhost:8080`.

### 3. Start the frontend

```bash
cd frontend
python3 -m http.server 5500
```

Open `http://localhost:5500/login.html`.

---

## Security Notes

A few things worth calling out explicitly:

- **GPS is extracted twice.** The frontend reads it for preview only. The backend ignores whatever coordinates the client claims and re-parses the raw image bytes. This prevents coordinate spoofing.
- **Roles come from the database, not the token.** Every role check hits the `profiles` table. JWT claims are not trusted for authorization decisions.
- **RLS is the last line of defense.** Even if the application layer has a bug, Postgres enforces access rules directly.
- **No secrets in the frontend.** The frontend only uses Supabase's anon key (safe for client-side) and reads from public endpoints.

---
