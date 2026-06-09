# Store Monitoring System

A FastAPI-based backend system that helps restaurant owners track how often their stores go offline during business hours. The system ingests store status polls, computes uptime/downtime per store, and exposes REST APIs to trigger and download detailed reports.

**Demo:** [Watch here](https://drive.google.com/file/d/1eAPfHd8sxxK5cD4mwecpveiBCv4wovN6/view?usp=drive_link) | **Sample Report:** [Download CSV](https://drive.google.com/file/d/16BcrmzFbiWU2eJQ3oplRGO0fpNRZO3nh/view?usp=sharing)

## Problem Overview

Restaurants in the US are expected to be online during their business hours. Due to various reasons, a store might go inactive for a few hours. Restaurant owners need a way to see how often this happened in the past.

This system:
- Ingests three data sources: store status polls (~hourly), store business hours, and store timezones
- Computes per-store uptime and downtime for the last hour, last day, and last week — only within business hours
- Exposes two APIs: one to trigger report generation and one to poll for completion and download the CSV

## Features

- **1.8M+ Records**: Efficiently handles large-scale store status data
- **Timezone-Aware**: Converts UTC poll timestamps to each store's local timezone before calculations
- **Business Hours Filtering**: Uptime is counted only within each store's operating hours
- **Async Report Generation**: Reports run in a background thread; clients poll for completion
- **CSV Export**: Download the final report as a structured CSV

## Tech Stack

- **Language:** Python 3.13+
- **Web Framework:** FastAPI
- **Database:** SQLite with SQLAlchemy ORM
- **Data Processing:** pandas
- **Background Tasks:** Python threading

## Getting Started

1. **Clone the repository and navigate to the project directory**

2. **Create and activate a virtual environment:**
   ```bash
   python -m venv venv
   venv\Scripts\activate        # Windows
   source venv/bin/activate     # macOS/Linux
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Load data into the database:**
   ```bash
   python load_data.py
   ```

5. **Start the server:**
   ```bash
   uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
   ```

6. **Access the interactive API docs:**
   Open http://localhost:8000/docs in your browser.

## API Endpoints

### POST /trigger_report
Starts report generation in the background and immediately returns a `report_id`.

**Response:**
```json
{
  "report_id": "uuid-string",
  "status": "Running",
  "message": "Report generation started"
}
```

### GET /get_report?report_id=\<report_id\>
Polls for report status or downloads the completed CSV.

**Still running:**
```json
{
  "report_id": "uuid-string",
  "status": "Running",
  "message": "Report generation in progress..."
}
```

**Completed:** Returns the CSV as a file download. In the Swagger UI (`/docs`), this appears as a download button.

## Data Schema

### Input

| File | Columns |
|------|---------|
| `store_status.csv` | `store_id`, `status` (active/inactive), `timestamp_utc` |
| `menu_hours.csv` | `store_id`, `dayOfWeek` (0=Mon, 6=Sun), `start_time_local`, `end_time_local` |
| `timezones.csv` | `store_id`, `timezone_str` (e.g. `America/Chicago`) |

**Defaults:** Stores missing from `menu_hours.csv` are assumed open 24×7. Stores missing from `timezones.csv` default to `America/Chicago`.

### Output

```
store_id, uptime_last_hour(in minutes), uptime_last_day(in hours), uptime_last_week(in hours),
downtime_last_hour(in minutes), downtime_last_day(in hours), downtime_last_week(in hours)
```

## Uptime/Downtime Calculation Logic

1. **Reference time**: The maximum timestamp in `store_status` is treated as "now" (since the dataset is static).
2. **Timezone conversion**: Each UTC poll timestamp is converted to the store's local timezone before any comparison with business hours.
3. **Business hours filtering**: Only polls that fall within the store's declared operating hours for that day of week are considered.
4. **Interpolation**: The store is assumed to hold its last known status until the next poll (last-observation-carried-forward). Total active time within business hours is summed as uptime; the remainder is downtime.
5. **Units**: Last-hour values are in minutes; last-day and last-week values are in hours.

## Architecture

### Database Models

| Model | Purpose |
|-------|---------|
| `StoreStatus` | Poll records — store_id, status, timestamp_utc |
| `BusinessHours` | Operating hours per store per day of week |
| `StoreTimezone` | Timezone string per store |
| `ReportStatus` | Tracks report job state and output file path |

### Core Modules

- `load_data.py` — One-time script to import all CSVs into SQLite
- `app/uptime_calculator.py` — Core logic: timezone conversion, business hours overlap, interpolation
- `app/report_generator.py` — Iterates all stores, calls the calculator, writes the CSV
- `app/background_tasks.py` — Runs report generation in a daemon thread and updates job status
- `app/main.py` — FastAPI app with `/trigger_report` and `/get_report` endpoints

## Performance Considerations

- **Chunked loading**: `load_data.py` reads the 1.8M-row CSV in 20K-row chunks to avoid memory spikes.
- **Caching**: Timezone strings and business hours are cached per store in `UptimeCalculator` to avoid repeated DB lookups.
- **Batch processing**: Stores are processed in batches of 50 during report generation.

## Improvement Ideas

- **Celery + Redis**: Replace Python threading with a proper task queue for better reliability and scalability.
- **PostgreSQL**: Move from SQLite to PostgreSQL for production-scale concurrent writes.
- **Redis cache**: Cache timezone and business hours at the application level to reduce DB load across multiple report runs.
- **Incremental reports**: Only recalculate stores whose poll data has changed since the last report run.

## Testing

```bash
python -m load_data                      # verify data loading
python -m app.test_uptime_calculator     # test calculation logic
python -m app.report_generator           # test report generation end-to-end
```
