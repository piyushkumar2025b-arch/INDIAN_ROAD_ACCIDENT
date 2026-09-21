# India Accident Hotspot Database & Visualization System

A production-grade Database Management System (DBMS) project combining a normalized **PostgreSQL** relational database (via **Supabase**), **Leaflet GIS** geospatial visualization, automated **Row Level Security (RLS)**, real **OpenStreetMap** geocoding, and temporal hotspot analytics for accident corridors across India.

---

## 1. Project Overview & Architecture

This system is built to ingest, store, manage, analyze, and visualize high-risk road accident locations ("hotspots") across Indian highways, expressways, and urban corridors. The architecture emphasizes **database integrity**, **strict normalization**, and **server-side processing** over ephemeral client-side calculations.

### Application Architecture Diagram

```text
+-------------------------------------------------------------------------------+
|                             CLIENT APPLICATION LAYER                          |
|                                                                               |
|   +--------------------------+           +--------------------------------+   |
|   |       Public User        |           |      Administrator Portal      |   |
|   |       (index.html)       |           |     (login.html / admin.html)  |   |
|   +------------+-------------+           +---------------+----------------+   |
|                |                                         |                    |
|                v                                         v                    |
|     +--------------------+                    +---------------------+         |
|     |  Leaflet GIS Map   |                    | Map Location Picker |         |
|     |  & Search Filters  |                    | & CSV Batch Ingest  |         |
|     +----------+---------+                    +----------+----------+         |
+----------------|-----------------------------------------|--------------------+
                 |                                         |
                 | HTTP / REST (Anon Key)                  | Authenticated JWT
                 v                                         v
+-------------------------------------------------------------------------------+
|                           SUPABASE / POSTGRESQL ENGINE                        |
|                                                                               |
|   +-----------------------------------------------------------------------+   |
|   |                     Row Level Security (RLS) Guard                    |   |
|   |          Public: Read Only   |   Admin Role: Full CRUD Operations      |   |
|   +-----------------------------------------------------------------------+   |
|                                                                               |
|   +--------------------------+   JOIN   +---------------------------------+   |
|   |        locations         +<---------+            accidents            |   |
|   |  (lat, lng, road, admin) |          |  (date, month, casualties, etc) |   |
|   +------------+-------------+          +----------------+----------------+   |
|                |                                         |                    |
|                v                                         v                    |
|   +--------------------------+                  +-------------------------+   |
|   | states / districts / rts |                  | types, causes, weather  |   |
|   +--------------------------+                  +-------------------------+   |
|                                                                               |
|   +-----------------------------------------------------------------------+   |
|   |                 Database Triggers & Stored Procedures                 |   |
|   |  - trg_derive_accident_month_year (Auto derives month & year)         |   |
|   |  - calculate_hotspot_score() (Empirical weighted scoring)             |   |
|   |  - check_potential_duplicate() (Collision duplicate detection)        |   |
|   +-----------------------------------------------------------------------+   |
|                                                                               |
|   +-----------------------------------------------------------------------+   |
|   |                       Analytical Database Views                       |   |
|   |  - hotspot_statistics (Spatial cluster scoring and tier ranking)      |   |
|   |  - monthly_accident_statistics (Seasonal and temporal trends)          |   |
|   |  - recent_accidents_feed (Optimized flattened administrative feed)    |   |
|   +-----------------------------------------------------------------------+   |
+-------------------------------------------------------------------------------+
```

---

## 2. Technology Stack

- **Frontend**: HTML5, CSS3, Vanilla JavaScript (ES6+ Modules, Zero bloated frameworks).
- **Mapping & GIS**: Leaflet.js v1.9.4, CartoDB Dark Matter / Voyager tiles, OpenStreetMap Nominatim Geocoding API with client debouncing.
- **Database & Backend**: PostgreSQL 15+ hosted on Supabase.
- **Authentication & Authorization**: Supabase Auth with custom `profiles` role-based access control (`admin` vs `viewer`).
- **Data Security**: PostgreSQL Row Level Security (RLS) policies on every table.

---

## 3. Academic DBMS Documentation

### 3.1 Normalization Process (1NF to 3NF)

The database schema is designed to eliminate update, insertion, and deletion anomalies.

- **First Normal Form (1NF)**: All attributes are atomic. Casualties are separated into distinct integer columns (`deaths`, `injured`, `critical_injuries`), and spatial coordinates are separated into `latitude` and `longitude` numeric values.
- **Second Normal Form (2NF)**: All non-key attributes are fully functionally dependent on the entire primary key. Geographic entities (States, Districts, Locations) are decoupled from transactional accident facts.
- **Third Normal Form (3NF)**: Transitive dependencies are removed:
  - Accident facts reference `location_id`.
  - `locations` references `district_id` and `state_id`.
  - `districts` references `state_id`.
  - Taxonomy strings (road types, collision types, causes, weather) are factored out into master lookup relations (`road_types`, `accident_types`, `accident_causes`, `weather_conditions`).

### 3.2 Relational Tables & Primary/Foreign Keys

1. **`states`**: Master reference table for all 28 Indian States and 8 Union Territories with official 2-letter codes.
2. **`districts`**: Administrative subdivisions partitioned by `state_id` foreign key with a composite unique constraint `(state_id, name)`.
3. **`road_types`**: Standard classifications (National Highway, State Highway, Expressway, City Arterial Road, Rural Road, Other).
4. **`accident_types`**: Collision taxonomy (Head-on, Rear-end, Side Impact, Pedestrian, Rollover, Motorcycle Crash, Multi-Vehicle Pileup).
5. **`accident_causes`**: Official causality categorizations (Overspeeding, Drunk Driving, Wrong-side Driving, Driver Fatigue, Distracted Driving, Poor Road, Mechanical Failure).
6. **`weather_conditions`**: Atmospheric context (Clear, Rain, Dense Fog, Smog, High Winds, Extreme Heat).
7. **`locations`**: Physical road corridor points containing latitude, longitude, road name, road number, and administrative references.
8. **`accidents`**: The central transactional fact table storing timestamps, casualty counts, vehicle units, severity, and audit references.
9. **`profiles`**: User authorization roles linked directly to `auth.users(id)`.

### 3.3 Database Constraints & Integrity Rules

- **Entity Integrity**: Every table uses a UUID primary key generated via `gen_random_uuid()`.
- **Referential Integrity**: 
  - `locations.state_id -> states.id` (`ON DELETE RESTRICT`)
  - `locations.district_id -> districts.id` (`ON DELETE RESTRICT`)
  - `accidents.location_id -> locations.id` (`ON DELETE CASCADE`)
- **Domain CHECK Constraints**:
  - `latitude BETWEEN -90.0 AND 90.0`
  - `longitude BETWEEN -180.0 AND 180.0`
  - `deaths >= 0`, `injured >= 0`, `critical_injuries >= 0`, `vehicles_involved >= 0`
  - `critical_injuries <= injured` (Logical constraint preventing impossible casualty counts)
  - `severity IN ('Minor', 'Moderate', 'Severe', 'Fatal')`
  - `month BETWEEN 1 AND 12`, `year BETWEEN 1990 AND 2100`

### 3.4 Automated Triggers

- **`trg_derive_accident_month_year`**: A `BEFORE INSERT OR UPDATE` trigger executing `derive_accident_month_year()`. It extracts `month` and `year` directly from `accident_date` on the database server. Users cannot manually enter inconsistent dates and months.
- **`trg_update_timestamp_*`**: Automatically updates `updated_at = NOW()` whenever records are modified.
- **`on_auth_user_created`**: Initializes a record in `profiles` whenever a new user signs up in `auth.users`.

### 3.5 Database Indexing Strategy

- **Foreign Key B-Trees**: Indexes created on `accidents(location_id)`, `locations(state_id)`, `locations(district_id)`, and `districts(state_id)` to eliminate full sequential scans during multi-table relational joins.
- **Temporal Indexes**: `accidents(accident_date DESC)` and composite index `accidents(year, month)` for rapid date-range filtering and annual safety reports.
- **Geospatial B-Tree**: Composite index on `locations(latitude, longitude)` for spatial bounding box filtering.
- **Attribute Filters**: Indexes on `accidents(severity)` and `accidents(verification_status)`.

### 3.6 Analytical Views

1. **`hotspot_statistics`**: Groups accidents by physical location, calculates total casualty metrics, determines dominant collision causes, computes highest risk months, and executes `calculate_hotspot_score()` to assign risk tiers.
2. **`accident_statistics_by_state`**: State-level casualty aggregation for national choropleth and ranking analysis.
3. **`monthly_accident_statistics`**: Temporal distribution across calendar months for identifying seasonal fog/monsoon surges.
4. **`cause_statistics`**: Frequency distribution and percentage shares across accident causality factors.
5. **`recent_accidents_feed`**: Pre-joined denormalized view designed for fast administrative inspection and sorting.

### 3.7 Calculated Accident Hotspot Score Methodology

The hotspot scoring algorithm evaluates clustered records at a given road point using an empirical weighted formula:

$$\text{Calculated Score} = \min\left(100, \text{round}\left(N_{\text{acc}} \times 2.5 + N_{\text{fatal}} \times 6.0 + D_{\text{total}} \times 7.0 + I_{\text{total}} \times 2.0 + R_{\text{bonus}} \times 3.0\right)\right)$$

Where:
- $N_{\text{acc}}$: Total recorded accidents at the location
- $N_{\text{fatal}}$: Count of accidents with $\ge 1$ death
- $D_{\text{total}}$: Sum of total deaths
- $I_{\text{total}}$: Sum of total injuries
- $R_{\text{bonus}}$: Recency factor (5 if last accident within 180 days, 3 within 365 days, else 1)

#### Risk Tiers
- **Critical Risk (Score 75 - 100)**: Immediate engineering and enforcement intervention needed.
- **High Risk (Score 50 - 74)**: Recurrent severe crashes and high casualty density.
- **Moderate Risk (Score 25 - 49)**: Frequent minor to moderate collisions.
- **Low Risk (Score < 25)**: Isolated incidents with low or zero fatalities.

---

## 4. Supabase Setup & Execution Guide

### Step 1: Create a Supabase Project
1. Log in to [Supabase](https://supabase.com) and click **New Project**.
2. Set your database password and choose your preferred geographic region (e.g., South Asia / Mumbai).

### Step 2: Execute SQL Scripts
Open the **SQL Editor** in your Supabase project dashboard. You have two execution options:

#### Option A: One-Click Execution (Recommended)
Copy the entire contents of:
`sql/all_in_one_setup.sql`
Paste into the Supabase SQL Editor and click **Run**. This will execute the entire DDL, reference seeds, views, functions, indexes, RLS policies, and demo benchmark data in one step!

#### Option B: Step-by-Step Execution Order
If you prefer running modular scripts, execute them strictly in this sequence:
1. `sql/schema.sql`
2. `sql/seed_reference_data.sql`
3. `sql/functions.sql`
4. `sql/views.sql`
5. `sql/indexes.sql`
6. `sql/rls.sql`
7. `sql/seed_sample_accidents.sql`

### Step 3: Configure Frontend Credentials
Open `js/config.js` and insert your project credentials:
```javascript
const CONFIG = {
  SUPABASE_URL: "https://your-project-id.supabase.co",
  SUPABASE_ANON_KEY: "eyJhbGciOi...",
  // ...
};
```
*(Find these in Supabase Dashboard -> **Project Settings** -> **API**)*

### Step 4: Create an Administrator User
1. In Supabase Dashboard, navigate to **Authentication** -> **Users** -> **Add User** -> **Create User**.
2. Enter an email (e.g. `admin@accidents.org`) and password.
3. Open the **SQL Editor** and promote this user to admin in the `profiles` table:
```sql
UPDATE public.profiles 
SET role = 'admin' 
WHERE email = 'admin@accidents.org';
```

---

## 5. Running the Application Locally

Since the frontend is built with pure HTML5, CSS3, and modern Vanilla JavaScript, you can host it using any static HTTP server:

```powershell
# Using Python
python -m http.server 8000

# Using Node.js npx serve
npx serve .
```
Then open your browser at `http://localhost:8000` (or the port indicated).

---

## 6. Features & User Guide

### 6.1 Public Interactive Map (`index.html`)
- **Interactive India Map**: Pan, zoom, and explore accident clusters across Indian states and national highway networks.
- **Section 9 Compliant Popups**: Clicking any hotspot displays:
  - Exact Location & Highway Corridor identifier
  - District and State
  - Total Accidents, Deaths, and Injuries
  - Highest Risk Month
  - Dominant Accident Cause
  - Road Classification
  - Calculated Accident Hotspot Score & Risk Badge
  - Last Recorded Accident Date
- **Live Filtering**: Filter simultaneously by State, District (cascading), Risk Tier, Month, Year, and Search term.
- **Bottom Insights Drawer**: Pull up the ranked list of dangerous corridors and the monthly seasonal trend graph.
- **DBMS Specification Modal**: Inspect the 3NF schema design, scoring formula, and RLS architecture directly from the top navigation.

### 6.2 Administrator Portal (`login.html` & `admin.html`)
- **Secure Authentication**: Supabase email/password login verified against `profiles.role = 'admin'`.
- **Database Telemetry Dashboard**: Real-time KPI cards for recorded accidents, fatalities, injuries, and physical corridor points.
- **Full CRUD Studio**: View paginated records, search incidents, edit existing reports, and delete records with confirmation safety checks.
- **Map-Based Location Picker**: Inside the Add/Edit form, search any town or highway with OpenStreetMap Nominatim geocoding or click directly on the interactive mini-map to auto-populate exact latitude and longitude coordinates.
- **CSV Data Ingestion Pipeline**: Drag-and-drop CSV datasets, validate schema headers, enforce coordinate ranges (-90 to 90, -180 to 180), verify casualty figures, and batch import valid rows into Supabase with detailed error logging.

---

## 7. Data Provenance & Methodology

Every accident fact record in the database includes `source_name`, `source_url`, and `verification_status`.
- **Public Datasets Referenced**: Ministry of Road Transport & Highways (MoRTH) Road Accidents in India annual publications, National Highways Authority of India (NHAI) black-spot engineering audits, and state traffic police open logs.
- **Demo Data Disclosure**: Initial benchmark sample records are explicitly labeled with `DEMO DATA — NOT REAL` in descriptions and source tags to adhere strictly to scientific data honesty rules.
