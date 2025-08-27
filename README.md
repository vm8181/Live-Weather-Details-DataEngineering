
# 🌦️ Weather Data Pipeline & Power BI Dashboard

## 📖 Project Overview
This project demonstrates a **modern BI architecture on Microsoft Fabric** for ingesting, transforming, and visualizing **real-time weather data**.  

We implement a **hybrid refresh approach**:
- **Automatic ingestion every 60 minutes** → ensures continuous **historical dataset**.  
- **On-demand refresh via a Power BI button** → users can fetch the **latest weather snapshot instantly**.  

The solution follows a **Medallion Architecture Bronze → Silver → Gold** design pattern, separating raw storage, cleaned append logs, and deduplicated reporting data.

---

## 🏗️ Architecture

### Components
1. **Crawler Notebook (Fabric Notebook)**  
   - Scrapes weather data for multiple cities.  
   - Saves raw CSV files into:  
     ```
     Files/weather_data_broze/weather_dataset {yyyy-MM-dd HH:mm:ss}.csv
     ```
   - Example: `weather_dataset 2025-08-27 10:15:30.csv`.

2. **Lakehouse (WeatherLakehouse)**  
   - Stores raw files in `Files/weather_data_broze`.  
   - Contains **Silver** and **Gold** Delta tables.

3. **Dataflow Gen2 (Weather Dataflow)**  
   - **Silver Table (`weather_data_silver`)**  
     - Appends all crawled datasets with lineage columns:  
       - `source_file`, `file_crawl_time`, `ingestion_time`.  
   - **Gold Table (`weather_data_gold`)**  
     - Deduplicated by `(city, date_time)`.  
     - Columns renamed for clarity (e.g. `temperature_c`, `humidity_pct`).  
     - Used directly by Power BI.

4. **Pipeline (WeatherHourlyPipeline)**  
   - Orchestrates:
     1. Run **Notebook (Data_Crawler)**.  
     2. **Wait 20–30 seconds** (file commit buffer).  
     3. Refresh **Dataflow Gen2**.  
   - Runs **every 60 minutes (schedule)**.  
   - Also exposed to Power BI button via **Power Automate**.

5. **Power BI Dashboard**  
   - Connects directly to `weather_data_gold` via **DirectLake**.  
   - Pages:
     - **Overview**: historical temperature/humidity trends by city.  
     - **Snapshot**: latest temperature/humidity by city.  
   - Includes **“Get Latest Data” button** (Power Automate → Run Pipeline).

---

## 🔄 Data Flow

```
Notebook (Crawler)
    ↓
Raw CSV files (Files/weather_data_broze in WeatherLakehouse)
    ↓
Dataflow Gen2
    ├─ Silver Table: weather_data_silver (append log)
    └─ Gold Table: weather_data_gold (deduped, reporting)
    ↓
Pipeline (orchestrates Notebook → Wait → Dataflow refresh)
    ↓
Power BI (DirectLake → Gold table)
    ↓
Users (historical insights + live button refresh)
```

---

## ⚙️ Pipeline Configuration

**Activities:**
1. **Data_Crawler (Notebook)**  
   - Timeout: `0:10:00`  
   - Retry: `2`  
   - Retry interval: `120 sec`

2. **Wait**  
   - Duration: `20–30 sec` (ensures files are committed in OneLake)

3. **Refresh_Weather_Dataflow (Dataflow Gen2)**  
   - Timeout: `0:20:00`  
   - Retry: `2`  
   - Retry interval: `120 sec`

**Schedule:** Every **60 minutes**, no concurrent runs.  

**Trigger:** Also exposed via Power Automate for **on-demand refresh**.

---

## 📊 Power BI Setup

### Connection
- Dataset: **weather_data_gold** (DirectLake from Lakehouse).  
- DirectLake ensures:
  - No dataset refresh required.  
  - Reports always show the current Gold table content.  

### Measures
```DAX
Latest Timestamp :=
MAX ( 'weather_data_gold'[date_time] )

Latest Temperature (°C) :=
VAR ts = [Latest Timestamp]
RETURN CALCULATE ( AVERAGE ( 'weather_data_gold'[temperature_c] ), 'weather_data_gold'[date_time] = ts )

Latest Humidity (%) :=
VAR ts = [Latest Timestamp]
RETURN CALCULATE ( AVERAGE ( 'weather_data_gold'[humidity_pct] ), 'weather_data_gold'[date_time] = ts )
```

### Pages
- **Overview**:  
  - Line chart → `date_time` vs `temperature_c` by `city`.  
  - Card visuals → latest metrics.  
- **Snapshot**:  
  - Table of latest weather by city (`Latest Timestamp`).  
  - Button → “Get Latest Data” (Power Automate).

---

## 🚀 Hybrid Refresh Approach

- **Hourly schedule**: Keeps history complete and up-to-date.  
- **On-demand button**: Fetches live weather instantly.  
- Ensures both:  
  - Historical dataset for analysis.  
  - Live, ad-hoc updates when needed.

---

## ✅ Validation & Monitoring

### SQL Checks (Lakehouse SQL Endpoint)
```sql
-- Latest record timestamp
SELECT MAX(date_time) AS latest_ts FROM weather_data_gold;

-- Coverage by city
SELECT city, COUNT(*) AS rows_, MIN(date_time) AS first_dt, MAX(date_time) AS last_dt
FROM weather_data_gold
GROUP BY city;

-- Check duplicates
SELECT city, date_time, COUNT(*) c
FROM weather_data_gold
GROUP BY city, date_time
HAVING COUNT(*) > 1;
```

### Monitoring
- **Pipeline run history** → check success/failures.  
- **Dataflow refresh history** → row counts.  
- **Power BI card** → shows `Latest Timestamp` for confidence.

---

## 🛠️ Best Practices

- **Keep raw files untouched** → always stored in `Files/weather_data_broze`.  
- **Use Silver for append logs** → preserves lineage.  
- **Use Gold for reporting** → deduped, clean, optimized.  
- **DirectLake for Power BI** → no dataset refresh needed.  
- **Retry & Timeout settings** → handle transient failures gracefully.  
- **Single source of refresh (Pipeline)** → avoid dedup errors.  

---

## 📌 Next Steps / Extensions

- Add **incremental refresh policy** in Dataflow Gen2 (e.g., keep 12 months, refresh last 3 days).  
- Extend crawler to fetch additional metrics (e.g., AQI, sunrise/sunset).  
- Add **alerts** (Teams/Email) when pipeline fails.  
- Integrate **real-time hub** → auto-run pipeline when new CSV lands.  
- Publish Power BI app with role-based access (city/region-level RLS).  

---

## 🗂️ Repository Structure
```
/WeatherProject
│
├─ /notebooks
│   └─ crawler_notebook.ipynb
│
├─ /dataflows
│   └─ weather_dataflow.json
│
├─ /pipelines
│   └─ WeatherHourlyPipeline.json
│
├─ /powerbi
│   └─ WeatherReport.pbix
│
└─ README.md   ← (this file)
```

---

💡 With this setup, you get **a robust end-to-end BI solution**:  
✔ Continuous history  
✔ Live updates on demand  
✔ Scalable Lakehouse storage  
✔ Seamless DirectLake reporting  
