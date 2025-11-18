# Air Quality Forecasting System - Lab Implementation

A complete machine learning system for forecasting PM2.5 air quality levels in Milan, Italy, using multiple sensor data and XGBoost regression.

## Overview

This system implements a batch ML pipeline that:
- Collects air quality data from 4 sensors across Milan
- Fetches weather forecasts from OpenMeteo API
- Trains an XGBoost model with lag features and weather data
- Generates daily predictions and monitors performance
- Publishes forecasts and hindcasts to a GitHub Pages dashboard

## Architecture

The pipeline follows a feature store architecture using Hopsworks:
- **Feature Groups**: `air_quality` (v1), `weather` (v2), `aq_predictions` (v2)
- **Feature View**: Joins air quality and weather features for training
- **Model Registry**: Stores trained XGBoost models with version control
- **Monitoring**: Tracks prediction accuracy via hindcast analysis

## Notebooks

### 1. Feature Backfill (`1_air_quality_feature_backfill.ipynb`)
Initial data loading and feature group creation.

**Key Steps:**
- Creates Hopsworks secrets for API keys (AQICN, sensor locations)
- Backfills historical PM2.5 data from AQICN API
- Backfills historical weather data from OpenMeteo
- Creates feature groups with data validation expectations

**Multi-Sensor Implementation:**
- Sensor configuration stored as JSON in Hopsworks secrets
- Supports multiple sensors via `SENSOR_LOCATION_JSON`:
```json
{
  "sensors": [
    {"city": "milan", "street": "lambrate", "country": "italy", "aqicn_url": "..."},
    {"city": "milan", "street": "Via Cristoforo Gandino", "country": "italy", "aqicn_url": "..."},
    ...
  ]
}
```

### 2. Daily Feature Pipeline (`2_air_quality_feature_pipeline.ipynb`)
Scheduled daily to collect fresh data.

**Key Steps:**
- Fetches today's PM2.5 readings from all 4 sensors
- Fetches 7-day weather forecast
- Inserts data into feature groups
- Runs via GitHub Actions at 6:11 AM UTC daily

**Multi-Sensor Handling:**
- Iterates through all sensors in configuration
- Collects PM2.5 data independently for each sensor
- Maintains sensor-specific lag features

### 3. Training Pipeline (`3_air_quality_training_pipeline.ipynb`)
Trains the XGBoost model with lag and weather features.

**Features Used:**
- **Weather features (5)**: temperature, precipitation, wind speed, wind direction, humidity
- **Lag features (3)**: PM2.5 from previous 1, 2, and 3 days
- **Target**: Next-day PM2.5 level

**Model Training:**
- Time-series split: Training data before 2025-05-01, test data after
- XGBoost regressor with default hyperparameters
- Model saved to Hopsworks Model Registry with metrics (MSE, R²)

**Multi-Sensor Training:**
- All sensor data used in training
- Model learns patterns across all 4 Milan sensors
- Predictions made per-sensor using sensor-specific lag features

### 4. Batch Inference (`4_air_quality_batch_inference.ipynb`)
Daily predictions and monitoring.

**Key Steps:**
1. Downloads latest model from Model Registry
2. Retrieves weather forecast for next 6 days
3. For each sensor:
   - Gets last 3 days of PM2.5 (lag features)
   - Generates predictions for next 6 days
4. Saves predictions to `aq_predictions` feature group
5. Creates forecast and hindcast visualizations
6. Uploads plots to GitHub Pages


**Multi-Sensor Predictions:**
- Individual predictions for each sensor (different lag features)
- City average computed by averaging across all 4 sensors
- Both individual and averaged predictions visualized

## Utility Functions (`mlfs/airquality/util.py`)

Key functions for the pipeline:

### Data Collection
- `get_pm25()`: Fetches PM2.5 data from AQICN API for a sensor
- `get_hourly_weather_forecast()`: Fetches weather forecast from OpenMeteo

### Visualization
- `plot_air_quality_forecast()`: Creates forecast/hindcast plots with AQI color bands
  - **Parameters**:
    - `city`, `street`: Location labels (use `'all_sensors'` for multi-sensor plots)
    - `df`: DataFrame with `predicted_pm25` (and optionally `pm25` for hindcast)
    - `file_path`: Where to save PNG
    - `hindcast`: Boolean to include actual PM2.5 overlay
  - **Output**: Logarithmic scale plot with AQI categories (Good, Moderate, Unhealthy, etc.)

### Backfill
- `backfill_predictions_for_monitoring()`: Creates synthetic hindcast data for initial testing

## Visualizations

The system generates 4 plots published to the GitHub Pages dashboard:

### 1. Forecast Plot (`pm25_forecast.png`)
Shows predicted PM2.5 for next 6 days across all 4 sensors.
- Red line with blue markers: Predicted PM2.5
- Colored bands: AQI categories
- Multiple lines overlay (one per sensor)
- Title: "PM2.5 Predicted (Logarithmic Scale) for milan, all_sensors"

### 2. City Average Forecast (`pm25_forecast_city_avg.png`)
Single averaged prediction across all sensors for cleaner visualization.
- Averaged predictions reduce noise from individual sensor variations
- Title: "PM2.5 Predicted (Logarithmic Scale) for milan, city_average"

### 3. Hindcast Plot (`pm25_hindcast_1day.png`)
Compares 1-day-ahead predictions vs actual measurements across all sensors.
- Red line: Predicted PM2.5 (from previous day)
- Black line with grey triangles: Actual PM2.5
- Shows model accuracy over time
- Multiple points per date (one per sensor)

### 4. City Average Hindcast (`pm25_hindcast_city_avg_1day.png`)
Averaged hindcast for cleaner performance monitoring.
- Single point per date (averaged across sensors)
- Easier to track overall model performance
- Reduces visual clutter

## GitHub Actions Workflow

**File**: `.github/workflows/air-quality-daily.yml`

**Schedule**: Daily at 6:11 AM UTC (`cron: '11 6 * * *'`)

**Steps:**
1. Checkout repository
2. Setup Python 3.10 with pip cache
3. Install dependencies from requirements.txt
4. Install Jupyter and mlfs-book kernel
5. Execute notebooks:
   - `2_air_quality_feature_pipeline.ipynb` (collect data)
   - `4_air_quality_batch_inference.ipynb` (predictions + plots)
6. Commit and push generated plots to `docs/air-quality/assets/img/`

**Required Secrets:**
- `HOPSWORKS_API_KEY`: Access to Hopsworks Feature Store
- `AQICN_API_KEY`: Access to air quality data API
- `SENSOR_LOCATION_JSON`: Sensor configuration in JSON format

## Dashboard

**URL**: Published via GitHub Pages at `docs/air-quality/index.md`

**Sections:**
1. **Forecast**:
   - Individual sensors forecast
   - City average forecast
2. **Model Performance Monitoring**:
   - Individual sensors hindcast (1-day ahead)
   - City average hindcast

## Multi-Sensor Design

The system was designed for scalability:

**Configuration-Driven:**
- Sensors defined in JSON (Hopsworks secret)
- Easy to add/remove sensors without code changes
- No hardcoded sensor names in pipeline code

**Data Model:**
- Primary key includes `street` column to distinguish sensors
- Each sensor has independent lag features
- Model learns from all sensors but predicts per-sensor

**Aggregation:**
- City-level metrics computed by averaging sensor predictions
- Provides both granular (per-sensor) and overview (city average) visualizations

**Current Deployment:**
- 4 sensors in Milan:
  - Lambrate
  - Via Cristoforo Gandino
  - Via Federico Chopin
  - Via Dei Lavoratori

## Key Features

1. **Lag Features**: Previous 3 days PM2.5 improves prediction accuracy by capturing recent trends
2. **Weather Integration**: Humidity, temperature, wind speed/direction as predictors
3. **Automated Monitoring**: Daily hindcast tracks model drift and prediction accuracy
4. **Multi-Scale Visualization**: Individual sensor and city average views for different use cases
5. **Production Ready**: Automated via GitHub Actions, no manual intervention needed
6. **Data Validation**: Great Expectations ensure data quality before insertion

## Performance Notes

**Model Limitations:**
- Struggles with sudden pollution spikes (e.g., Nov 15 spike: predicted 26.7, actual 152)
- Works well for stable periods with gradual changes
- May under-predict during pollution events (regression to mean behavior)
- Lag features reflect past conditions, can't predict unprecedented events

## Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HOPSWORKS_API_KEY="your_key"
export AQICN_API_KEY="your_key"
export SENSOR_LOCATION_JSON='{"sensors": [...]}'

# Run notebooks in order
jupyter notebook notebooks/airquality/

# Or run via command line
jupyter nbconvert --to notebook --execute notebooks/airquality/2_air_quality_feature_pipeline.ipynb
jupyter nbconvert --to notebook --execute notebooks/airquality/4_air_quality_batch_inference.ipynb
```

## Technical Changes Made

### Util.py Enhancements
- `plot_air_quality_forecast()` supports both single sensor and multi-sensor visualizations
- Flexible `street` parameter accepts sensor names, 'all_sensors', or 'city_average'
- Hindcast mode overlays actual vs predicted values

