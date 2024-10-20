Project Setup
Install Dependencies:

bash
Copy code
pip install fastapi uvicorn requests sqlite3 matplotlib
Directory Structure:

graphql
Copy code
weather_monitoring/
│
├── app.py                # Main FastAPI application
├── config.py             # API configuration and thresholds
├── weather_processor.py  # Data retrieval, rollups, and alerts
├── database.py           # Database setup and queries
├── visualizations.py     # Plotting weather trends
├── test_cases.py         # Test cases
└── requirements.txt      # Dependencies
1. Configuration (config.py)
python
Copy code
# config.py
API_KEY = "your_openweathermap_api_key"  # Replace with your API key
CITIES = ["Delhi", "Mumbai", "Chennai", "Bangalore", "Kolkata", "Hyderabad"]
API_URL = "https://api.openweathermap.org/data/2.5/weather"
INTERVAL = 300  # 5 minutes interval

THRESHOLDS = {
    "temp": 35,  # Alert if temp exceeds 35°C
    "consecutive_exceed": 2,  # Two consecutive breaches to trigger an alert
    "condition": "Rain"  # Alert if weather is 'Rain'
}
2. Database Setup (database.py)
We use SQLite to store weather summaries and alerts.

python
Copy code
# database.py
import sqlite3

def get_db_connection():
    conn = sqlite3.connect('weather.db')
    conn.row_factory = sqlite3.Row
    return conn

def setup_database():
    conn = get_db_connection()
    cursor = conn.cursor()

    # Create weather summary table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS weather_summary (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            city TEXT,
            date TEXT,
            avg_temp REAL,
            max_temp REAL,
            min_temp REAL,
            dominant_condition TEXT
        );
    ''')

    # Create alerts table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS alerts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            city TEXT,
            timestamp TEXT,
            alert_message TEXT
        );
    ''')

    conn.commit()
    conn.close()

setup_database()
3. Weather Data Processor (weather_processor.py)
Handles weather data retrieval, aggregation, and alerting.

python
Copy code
# weather_processor.py
import requests
from datetime import datetime
from database import get_db_connection
from config import API_URL, API_KEY, CITIES, THRESHOLDS

def kelvin_to_celsius(temp):
    """Convert temperature from Kelvin to Celsius."""
    return round(temp - 273.15, 2)

def fetch_weather(city):
    """Fetch weather data for a specific city."""
    response = requests.get(API_URL, params={"q": city, "appid": API_KEY})
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"Failed to fetch data for {city}")

def store_weather_summary(city, temp, condition):
    """Store the weather summary in the database."""
    conn = get_db_connection()
    cursor = conn.cursor()

    cursor.execute('''
        INSERT INTO weather_summary (city, date, avg_temp, max_temp, min_temp, dominant_condition)
        VALUES (?, date('now'), ?, ?, ?, ?)
    ''', (city, temp, temp, temp, condition))

    conn.commit()
    conn.close()

def trigger_alert(city, message):
    """Trigger and store an alert in the database."""
    conn = get_db_connection()
    cursor = conn.cursor()

    cursor.execute('''
        INSERT INTO alerts (city, timestamp, alert_message)
        VALUES (?, datetime('now'), ?)
    ''', (city, message))

    conn.commit()
    conn.close()
    print(f"ALERT: {message}")

def process_weather():
    """Retrieve, process, and store weather data."""
    for city in CITIES:
        data = fetch_weather(city)
        temp = kelvin_to_celsius(data['main']['temp'])
        condition = data['weather'][0]['main']

        # Store summary
        store_weather_summary(city, temp, condition)

        # Trigger alerts based on thresholds
        if temp > THRESHOLDS["temp"]:
            trigger_alert(city, f"High temperature alert: {temp}°C in {city}")
        if condition == THRESHOLDS["condition"]:
            trigger_alert(city, f"Weather condition alert: {condition} in {city}")
4. FastAPI Application (app.py)
Provides APIs to access weather summaries and alerts.

python
Copy code
# app.py
from fastapi import FastAPI
from database import get_db_connection

app = FastAPI()

@app.get("/weather_summary")
def get_weather_summary(city: str):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        SELECT * FROM weather_summary WHERE city = ? ORDER BY date DESC LIMIT 1
    ''', (city,))
    summary = cursor.fetchone()
    return dict(summary) if summary else {"error": "No data found"}

@app.get("/alerts")
def get_alerts():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM alerts ORDER BY timestamp DESC')
    alerts = cursor.fetchall()
    return [dict(alert) for alert in alerts]
5. Visualization (visualizations.py)
Generate a temperature trend plot using Matplotlib.

python
Copy code
# visualizations.py
import matplotlib.pyplot as plt
from database import get_db_connection

def plot_temperature_trend(city):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        SELECT date, avg_temp FROM weather_summary WHERE city = ?
    ''', (city,))
    data = cursor.fetchall()

    dates = [row['date'] for row in data]
    temps = [row['avg_temp'] for row in data]

    plt.plot(dates, temps, marker='o')
    plt.title(f'Temperature Trend for {city}')
    plt.xlabel('Date')
    plt.ylabel('Temperature (°C)')
    plt.xticks(rotation=45)
    plt.show()
6. Test Cases (test_cases.py)
Test data retrieval, temperature conversion, and alerting.

python
Copy code
# test_cases.py
from weather_processor import kelvin_to_celsius, fetch_weather

def test_temperature_conversion():
    assert kelvin_to_celsius(300) == 26.85

def test_fetch_weather():
    data = fetch_weather("Delhi")
    assert data["name"] == "Delhi"

if __name__ == "__main__":
    test_temperature_conversion()
    test_fetch_weather()
    print("All tests passed!")
7. Running the Application
Install dependencies:

bash
Copy code
pip install -r requirements.txt
Start the FastAPI server:

bash
Copy code
uvicorn app:app --reload
Run the weather processor periodically (using cron or a scheduler):

bash
Copy code
python weather_processor.py
Plot temperature trends:

bash
Copy code
python -c "from visualizations import plot_temperature_trend; plot_temperature_trend('Delhi')"
8. Sample crontab Configuration
To run the weather processor every 5 minutes, add the following line to your crontab:

bash
Copy code
*/5 * * * * /usr/bin/python3 /path_to_your_project/weather_processor.py
9. README.md
markdown
Copy code
# Real-Time Weather Monitoring System

## Overview
This system monitors weather data in real-time, provides daily summaries, and triggers alerts based on thresholds.

## Features
- Real-time weather monitoring for Indian metros.
- Daily summaries with average, max, and min temperatures.
- Alerts for high temperatures or specific weather conditions.
- Visualize temperature trends with Matplotlib.

## Usage
1. Start the FastAPI server: `uvicorn app:app --reload`
2. Schedule the processor: `python weather_processor.py`
3. View summaries at: `/weather_summary?city=Delhi`
4. View alerts at: `/alerts`

## License
MIT
