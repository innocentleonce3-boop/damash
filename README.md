

        
# FILE: server.py
# Run command: python server.py

import sqlite3
import asyncio
from datetime import datetime
from fastapi import FastAPI, HTTPException, Header, Request
from fastapi.responses import HTMLResponse
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import uvicorn

# --- 1. DATABASE SETUP ---
def init_db():
    conn = sqlite3.connect('damash_data.db')
    c = conn.cursor()
    # Create table if not exists
    c.execute('''CREATE TABLE IF NOT EXISTS sensor_logs
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  timestamp TEXT, 
                  device_id TEXT, 
                  temperature REAL, 
                  humidity REAL, 
                  soil_moisture INTEGER,
                  alert_status TEXT)''')
    conn.commit()
    conn.close()

init_db()

# --- 2. APPLICATION SETUP ---
app = FastAPI(title="Damash AI System")

# Allow ESP32 to connect
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# --- 3. DATA MODEL ---
class SensorData(BaseModel):
    device_id: str
    temperature: float
    humidity: float
    soil_moisture: int

# --- 4. AI ANALYSIS ENGINE ---
def analyze_data(temp, hum, soil):
    """Core AI Logic: Analyzes sensor values for risks."""
    alerts = []
    
    # Temperature Logic
    if temp > 45.0:
        alerts.append("🔥 CRITICAL: High Temperature")
    elif temp > 35.0:
        alerts.append("⚠️ WARNING: Temperature Rising")

    # Humidity Logic
    if hum > 80.0:
        alerts.append("💧 WARNING: High Humidity Risk")
    
    # Soil Logic (Assuming 0-100 scale)
    if soil < 20:
        alerts.append("🌱 CRITICAL: Low Soil Moisture (Water Needed)")
        
    if not alerts:
        return "✅ Status: Normal"
    
    return " | ".join(alerts)

# --- 5. API ENDPOINT (ESP32 sends data here) ---
@app.post("/api/sensors")
async def receive_data(data: SensorData):
    # Run AI Analysis
    status = analyze_data(data.temperature, data.humidity, data.soil_moisture)
    
    # Save to Database
    conn = sqlite3.connect('damash_data.db')
    c = conn.cursor()
    c.execute("INSERT INTO sensor_logs (timestamp, device_id, temperature, humidity, soil_moisture, alert_status) VALUES (?,?,?,?,?,?)",
              (datetime.now().strftime("%Y-%m-%d %H:%M:%S"), data.device_id, data.temperature, data.humidity, data.soil_moisture, status))
    conn.commit()
    conn.close()
    
    print(f"Data Received: {data.temperature}°C | Status: {status}")
    return {"status": "success", "analysis": status}

# --- 6. LIVE DASHBOARD (View in Browser) ---
@app.get("/", response_class=HTMLResponse)
async def dashboard(request: Request):
    conn = sqlite3.connect('damash_data.db')
    c = conn.cursor()
    # Get last 20 readings
    c.execute("SELECT * FROM sensor_logs ORDER BY id DESC LIMIT 20")
    rows = c.fetchall()
    conn.close()
    
    # Generate HTML
    html = """
    <html>
    <head>
        <title>Damash AI Dashboard</title>
        <meta http-equiv="refresh" content="3"> <!-- Auto Refresh -->
        <style>
            body { font-family: 'Segoe UI', sans-serif; background: #f4f4f9; padding: 20px; }
            h1 { color: #2c3e50; }
            table { width: 100%; border-collapse: collapse; background: white; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
            th, td { padding: 15px; text-align: left; border-bottom: 1px solid #ddd; }
            th { background: #2c3e50; color: white; }
            .alert { background-color: #ffdddd; color: #d8000c; font-weight: bold; }
            .normal { color: #4caf50; }
        </style>
    </head>
    <body>
        <h1>🔥 Damash AI Live Monitoring</h1>
        <p>Last updated: <b>""" + datetime.now().strftime("%H:%M:%S") + """</b></p>
        <table>
            <tr>
                <th>Time</th>
                <th>Device ID</th>
                <th>Temp (°C)</th>
                <th>Humidity (%)</th>
                <th>Soil (%)</th>
                <th>AI Analysis</th>
            </tr>
    """
    
    for row in reversed(rows): # Show newest first
        css_class = "alert" if "CRITICAL" in row[6] else ""
        html += f"""
            <tr class="{css_class}">
                <td>{row[1]}</td>
                <td>{row[2]}</td>
                <td>{row[3]}</td>
                <td>{row[4]}</td>
                <td>{row[5]}</td>
                <td>{row[6]}</td>
            </tr>
        """
    
    html += "</table></body></html>"
    return HTMLResponse(content=html)

# --- 7. START SERVER ---
if __name__ == "__main__":
    print("🚀 Damash AI Server Starting...")
    print("📱 Dashboard: http://localhost:8000")
    uvicorn.run(app, host="0.0.0.0", port=8000)
 /*
 * DAMASH AI - FIRMWARE
 * Hardware: ESP32 + DHT22 + Soil Sensor
 */

#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <DHT.h>

// ================= CONFIGURATION =================
const char* WIFI_SSID = "YOUR_WIFI_NAME";       // CHANGE THIS
const char* WIFI_PASS = "YOUR_WIFI_PASSWORD";   // CHANGE THIS
// Use your computer's IP address if running server locally, or public IP if cloud
const char* SERVER_URL = "http://192.168.1.100:8000/api/sensors"; 

// ================= PINS =================
#define DHTPIN 4          // Pin connected to DHT22 Data
#define SOIL_PIN 34       // Pin connected to Soil Sensor Analog
#define BUZZER_PIN 25     // Pin connected to Buzzer
#define LED_PIN 2         // Built-in LED

#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// Variables
unsigned long lastTime = 0;
unsigned long timerDelay = 5000; // Send data every 5 seconds

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  // Connect to WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.println("Connecting to WiFi...");
  
  while(WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    digitalWrite(LED_PIN, !digitalRead(LED_PIN)); // Blink LED while connecting
  }
  
  Serial.println("");
  Serial.print("Connected! IP Address: ");
  Serial.println(WiFi.localIP());
  digitalWrite(LED_PIN, HIGH); // Solid LED when connected
}

void loop() {
  // Send data every 'timerDelay' milliseconds
  if ((millis() - lastTime) > timerDelay) {
    // Check WiFi connection
    if(WiFi.status()== WL_CONNECTED){
      digitalWrite(LED_PIN, HIGH);
      
      // 1. READ SENSORS
      float temp = dht.readTemperature();
      float hum = dht.readHumidity();
      int soilRaw = analogRead(SOIL_PIN);
      
      // Check if reads failed
      if (isnan(temp) || isnan(hum)) {
        Serial.println("Failed to read from DHT sensor!");
        lastTime = millis();
        return;
      }

      // Convert soil to percentage (Assume 0-4095 dry to wet, adjust logic as needed)
      int soilPercent = map(soilRaw, 0, 4095, 0, 100);

      Serial.printf("Temp: %.1f°C | Hum: %.1f%% | Soil: %d%%\n", temp, hum, soilPercent);

      // 2. LOCAL AI LOGIC (Edge Warning)
      if(temp > 50.0) {
        triggerAlarm(); // Local alert
      }

      // 3. SEND TO CLOUD
      sendToServer(temp, hum, soilPercent);
      
      digitalWrite(LED_PIN, LOW);
    } else {
      Serial.println("WiFi Disconnected");
    }
    lastTime = millis();
  }
}

void sendToServer(float t, float h, int s) {
  HTTPClient http;
  http.begin(SERVER_URL);
  http.addHeader("Content-Type", "application/json");
  
  // Create JSON
  StaticJsonDocument<200> doc;
  doc["device_id"] = "ESP32_NODE_01";
  doc["temperature"] = t;
  doc["humidity"] = h;
  doc["soil_moisture"] = s;
  
  char jsonOutput[256];
  serializeJson(doc, jsonOutput);
  
  int httpCode = http.POST(jsonOutput);
  
  if (httpCode > 0) {
    String payload = http.getString();
    Serial.print("Server Response: ");
    Serial.println(payload);
  } else {
    Serial.print("Error code: ");
    Serial.println(httpCode);
  }
  http.end();
}

void triggerAlarm() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(500);
  digitalWrite(BUZZER_PIN, LOW);
}
