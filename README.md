
// Blynk Configuration
#define BLYNK_TEMPLATE_ID "TMPL3qo_g_HOr"
#define BLYNK_TEMPLATE_NAME "Health Monitoring"
#define BLYNK_AUTH_TOKEN "QC9XpMgr6Bzr4zTXd1kMpJwiXIm8GaOG"

// Include Libraries
#include <Wire.h>
#include "MAX30100_PulseOximeter.h"
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// Define Reporting Intervals
#define REPORTING_PERIOD_MS 1000     // Heart rate and SpO2
#define DHT_READ_INTERVAL 5000       // Temp & Humidity every 5 sec
#define DS18B20_READ_INTERVAL 2000  // DS18B20 read interval

// DHT11 Setup
#define DHTPIN D3        // Use a pin not used by I2C (D3 = GPIO0)
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// DS18B20 Setup (1-Wire)
#define ONE_WIRE_BUS D4   // GPIO2 (change if needed)
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// WiFi Credentials
char ssid[] = "YourWiFiSSID";
char pass[] = "YourWiFiPassword";

// MAX30100 Setup
PulseOximeter pox;
uint32_t tsLastReport = 0;
uint32_t lastDHTRead = 0;
uint32_t lastDS18B20Read = 0;  // New timing for DS18B20

// Virtual Pins in Blynk
#define VPIN_BPM   V4
#define VPIN_SPO2  V3
#define VPIN_TEMP  V0
#define VPIN_HUM   V1
#define VPIN_DS18_TEMP V2 // DS18B20 Temperature

void onBeatDetected() {
  Serial.println("💓 Beat detected!");
}

void setup() {
  Serial.begin(115200);
  
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
  dht.begin();
  sensors.begin(); // Start DS18B20 sensor

  // Initialize MAX30100
  Serial.println("Initializing MAX30100...");
  if (!pox.begin()) {
    Serial.println("❌ Failed to initialize MAX30100");
    while (1); // Stop here if MAX30100 doesn't initialize
  } else {
    Serial.println("✅ MAX30100 initialized successfully.");
  }

  pox.setOnBeatDetectedCallback(onBeatDetected);
}

void loop() {
  Blynk.run();
  pox.update();

  // Heart Rate & SpO2 - Every REPORTING_PERIOD_MS (1 second)
  if (millis() - tsLastReport > REPORTING_PERIOD_MS) {
    tsLastReport = millis();

    float bpm = pox.getHeartRate();
    float spo2 = pox.getSpO2();

    // Check if valid readings are available
    if (bpm > 0 && spo2 > 0) {
      Serial.print("❤️ Heart rate: ");
      Serial.print(bpm);
      Serial.print(" bpm | SpO2: ");
      Serial.print(spo2);
      Serial.println(" %");

      Blynk.virtualWrite(VPIN_BPM, bpm);
      Blynk.virtualWrite(VPIN_SPO2, spo2);
    } else {
      // If Heart Rate and SpO2 are invalid, print an error
      Serial.println("⚠️ Invalid Heart Rate or SpO2 values. Please check MAX30100 connection.");
    }
  }

  // Temperature & Humidity from DHT11 - Every DHT_READ_INTERVAL (5 seconds)
  if (millis() - lastDHTRead > DHT_READ_INTERVAL) {
    lastDHTRead = millis();

    float temp = dht.readTemperature();
    float hum = dht.readHumidity();

    if (isnan(temp) || isnan(hum)) {
      Serial.println("⚠️ Failed to read from DHT11");
    } else {
      Serial.print("🌡️ Temp (DHT11): ");
      Serial.print(temp);
      Serial.print(" °C | 💧 Humidity: ");
      Serial.print(hum);
      Serial.println(" %");

      Blynk.virtualWrite(VPIN_TEMP, temp);
      Blynk.virtualWrite(VPIN_HUM, hum);
    }
  }

  // Temperature from DS18B20 - Every DS18B20_READ_INTERVAL (2 seconds)
  if (millis() - lastDS18B20Read > DS18B20_READ_INTERVAL) {
    lastDS18B20Read = millis();

    sensors.requestTemperatures(); // Get temperature from DS18B20
    float ds18Temp = sensors.getTempCByIndex(0);  // Get temperature from first DS18B20

    if (ds18Temp != DEVICE_DISCONNECTED_C) {
      Serial.print("🌡️ Temp (DS18B20): ");
      Serial.print(ds18Temp);
      Serial.println(" °C");

      Blynk.virtualWrite(VPIN_DS18_TEMP, ds18Temp);  // Send DS18B20 temp to Blynk
    } else {
      Serial.println("⚠️ DS18B20 sensor not connected!");
    }
  }
}
