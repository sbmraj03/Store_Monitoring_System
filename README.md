DEMO URL => https://drive.google.com/file/d/1eAPfHd8sxxK5cD4mwecpveiBCv4wovN6/view?usp=drive_link


# Store Monitoring System

A FastAPI-based system for monitoring restaurant store uptime and generating detailed reports.

## CSV_OUTPUT_REPORT
https://drive.google.com/file/d/16BcrmzFbiWU2eJQ3oplRGO0fpNRZO3nh/view?usp=sharing


## Important Features

- **Data Processing**: Handles 1.8M+ store status records
- **Timezone Handling**: Supports multiple US timezones
- **Business Hours**: Calculates uptime only during operational hours
- **REST APIs**: FastAPI endpoints for report generation
- **CSV Export**: Downloadable reports in CSV format
- **Database Storage**: SQLite database for efficient data querying

## Tech Stack Used
- **Language:** Python 3.13+
- **Web Framework:** FastAPI
- **Database:** SQLite (SQLAlchemy ORM)
- **Data Processing:** pandas
- **Background Tasks:** Threading for async report generation


## Getting Started


1. **First Clone the repository and navigate to the project directory:**

2. **Create and activate virtual environment:**
   
   ```bash
   python -m venv venv
   venv\Scripts\activate
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

2. **Access the API:**
   Open http://localhost:8000/docs with your browser to see the interactive Swagger UI.
### API Endpoints

#### 1. Trigger Report Generation
```http
POST /trigger_report
```
Starts report generation process.

**Response:**
```json
{
  "report_id": "uuid-string",
  "status": "Running",
  "message": "Report generation started",
}
```

#### 2. Get Report Status/Download
```http
GET /get_report?report_id=<report_id>
```

***Responses:***
1. **Report still running:**
```json
{
  "report_id": "uuid-string",
  "status": "Running",
  "message": "Report generation in progress..."
}
```
2. **Report completed:**

> In FastAPI docs (`/docs`), you will see that the report has appeared as a **download button** 


## Data Schema

### Input Data
1. **store_status.csv**
   - `store_id`: Unique store identifier
   - `status`: 'active' or 'inactive'
   - `timestamp_utc`: UTC timestamp

2. **business_hours.csv**
   - `store_id`: Unique store identifier
   - `dayOfWeek`: 0=Monday, 6=Sunday
   - `start_time_local`: Local opening time
   - `end_time_local`: Local closing time

3. **store_timezones.csv**
   - `store_id`: Unique store identifier
   - `timezone_str`: Timezone (e.g., 'America/Chicago')


### Output Report Schema
```csv
store_id, uptime_last_hour(in minutes), uptime_last_day(in hours), update_last_week(in hours), downtime_last_hour(in minutes), downtime_last_day(in hours), downtime_last_week(in hours) 
```


## Architecture

### Database Models
- **StoreStatus**: Stores poll data (active/inactive timestamps)
- **BusinessHours**: Store operating hours by day of week
- **StoreTimezone**: Store timezone information
- **ReportStatus**: Report generation tracking

### Core Components
- **UptimeCalculator**: Core business logic for uptime/downtime calculations
- **FastAPI App**: REST API endpoints
- **Report Generator**: CSV report creation
- **Database Layer**: SQLAlchemy ORM with SQLite

## Hours Overlap & Uptime/Downtime Calculation Logic

Here is how I calculated uptime and downtime for each store for the last hour, last day, and last week:

- **Time Interval Calculation**: I checked the store status for the time I want (like last hour, last day, or last week).  
- **Business Hours Handling**: I only count time that is inside the store’s business hours.  
  - If a store’s business hours go past midnight (like 10 PM to 2 AM), I handle it correctly across the two days.  
- **Status Interpolation**: I assumed that the store stays in the last known status until the next poll.  
- **Timezone Awareness**: All UTC timestamps are converted to the store’s local time before calculating uptime.  
- **Caching for Efficiency**: I saved timezone and business hours for each store so I don’t need to ask the database many times.  
- **Calculation Method**:
  1. Find total business hours in the interval.  
  2. Add up the time the store was active → this is uptime.  
  3. Downtime = total business hours − uptime.  
- **Unit Conversion**:
  - Last hour → minutes  
  - Last day & last week → hours


## Performance Considerations

- **Batch Processing**: Processed stores in batches so that the program doesn’t use too much memory at once.  
- **Caching**: Saved timezone and business hours for each store so we don’t have to ask the database again and again.  
- **Progress Tracking**: The program shows how many stores have been processed while generating big reports.

## Improvement Ideas

- We can use Celery/Redis for asynchronous report generation to handle large datasets more efficiently.  

- We can move from SQLite to PostgreSQL to achieve better performance at scale.  

- Can implement a Redis cache for frequently accessed data to reduce repeated computations and speed up report generation.



## Testing

1. Check data loading: `python -m load_data`
2. Test uptime calculation: `python -m app.test_uptime_calculator`
3. Test report generation: `python -m app.report_generator`


