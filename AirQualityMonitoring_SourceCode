#define BLYNK_TEMPLATE_ID "TMPL64Ikdu3hT"
#define BLYNK_TEMPLATE_NAME "AirPollutionMonitoring"

#define BLYNK_PRINT Serial
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

char auth[] = "Hi0uKUuPTKa3GxeCeRstlqmFMOFQGwPZ"; // Blynk authentication token
char ssid[] = "MCM DOMAIN"; // WiFi network SSID
char pass[] = "VivaMCM!"; // WiFi network password

BlynkTimer timer; // Blynk timer object

int gasPin = 34; // Pin for gas sensor
int methanePin = 35; // Pin for methane sensor
int coPin = 32; // Pin for CO sensor

#define DHTPIN 4 // Pin for DHT11 sensor
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE); // DHT sensor object

LiquidCrystal_I2C lcd(0x27, 16, 2); // LCD object with I2C address 0x27, 16x2 characters

int displayIndex = 0; // Index for rotating display on LCD
#define WINDOW_SIZE 5 // Size of sliding window for sensor readings
int sensorReadings[WINDOW_SIZE]; // Array to store sensor readings

// Hysteresis thresholds and states for each sensor
struct Hysteresis {
    int thresholdHigh;
    int thresholdLow;
    bool alertState;
};

Hysteresis gasHysteresis = {550, 200, false}; // Gas sensor hysteresis thresholds
Hysteresis methaneHysteresis = {55, 0, false}; // Methane sensor hysteresis thresholds
Hysteresis coHysteresis = {1000, 500, false}; // CO sensor hysteresis thresholds

void checkHysteresis(int value, Hysteresis& sensorHysteresis, int vPin) {
    if (!sensorHysteresis.alertState && value >= sensorHysteresis.thresholdHigh) {
        sensorHysteresis.alertState = true;
        Blynk.virtualWrite(vPin, "Alert: High Level Detected");

        

        // Log event to Blynk
        Blynk.logEvent("pollution_alert", "Bad Air Quality Detected");
    } else if (sensorHysteresis.alertState && value <= sensorHysteresis.thresholdLow) {
        sensorHysteresis.alertState = false;
        Blynk.virtualWrite(vPin, "Alert: Level Normalized");

    }
}

float calculateSMA() {
    float sum = 0;
    for (int i = 0; i < WINDOW_SIZE; i++) {
        sum += sensorReadings[i];
    }
    return sum / WINDOW_SIZE;
}

void updateSensorReadings(int newValue) {
    for (int i = WINDOW_SIZE - 1; i > 0; i--) {
        sensorReadings[i] = sensorReadings[i - 1];
    }
    sensorReadings[0] = newValue;
}

void sendSensorDataWithSMA() {
    int newReading = analogRead(gasPin);
    updateSensorReadings(newReading);
    float smaValue = calculateSMA();

    checkHysteresis(newReading, gasHysteresis, V10);

    Blynk.virtualWrite(V5, smaValue); // Sending SMA value to Blynk

    float humidity = dht.readHumidity();
    float temperature = dht.readTemperature();

    if (isnan(humidity) || isnan(temperature)) {
        Serial.println("Failed to read from DHT sensor!");
        return;
    }

    int methaneValue = analogRead(methanePin);
    int coValue = analogRead(coPin);

    checkHysteresis(methaneValue, methaneHysteresis, V11);
    checkHysteresis(coValue, coHysteresis, V12);

    Blynk.virtualWrite(V1, temperature);
    Blynk.virtualWrite(V2, humidity);
    Blynk.virtualWrite(V0, newReading);
    Blynk.virtualWrite(V3, methaneValue);
    Blynk.virtualWrite(V4, coValue);

    Serial.print("Temperature: "); Serial.println(temperature);
    Serial.print("Humidity: "); Serial.println(humidity);
    Serial.print("Gas: "); Serial.println(newReading);
    Serial.print("Methane: "); Serial.println(methaneValue);
    Serial.print("CO: "); Serial.println(coValue);
}

void displaySensorData() {
    float humidity = dht.readHumidity();
    float temperature = dht.readTemperature();
    int gasValue = analogRead(gasPin);
    int methaneValue = analogRead(methanePin);
    int coValue = analogRead(coPin);

    lcd.clear();

    switch (displayIndex) {
        case 0: 
            lcd.setCursor(0, 0);
            lcd.print("Temp: " + String(temperature) + " C");
            break;
        case 1: 
            lcd.setCursor(0, 0);
            lcd.print("Hum: " + String(humidity) + "%");
            break;
        case 2: 
            lcd.setCursor(0, 0);
            lcd.print("Methane: " + String(methaneValue));
            break;
        case 3: 
            lcd.setCursor(0, 0);
            lcd.print("CO: " + String(coValue));
            break;
        case 4: 
            lcd.setCursor(0, 0);
            lcd.print("Gas: " + String(gasValue));
            break;
        default: 
            displayIndex = 0;
            break;
    }

    // Determine air quality status
    bool airQualityBad = (gasHysteresis.alertState || methaneHysteresis.alertState || coHysteresis.alertState);

    // Display air quality status on LCD
    lcd.setCursor(0, 1);
    if (airQualityBad) {
        lcd.print("Air Quality: Bad");
    } else {
        lcd.print("Air Quality:Good");
    }

    displayIndex = (displayIndex + 1) % 5;
}



void setup() {
    Serial.begin(115200); // Start serial communication
    while (!Serial); // Wait for Serial Monitor to open

    // Initialize Blynk
    Blynk.begin(auth, ssid, pass);

    // Initialize DHT sensor
    dht.begin();

    // Initialize LCD
    lcd.init();
    lcd.backlight();

    // Initialize sensor readings array
    for (int i = 0; i < WINDOW_SIZE; i++) {
        sensorReadings[i] = 0;
    }

    // Set up Blynk timer intervals for sensor data, display, and pollution alert
    timer.setInterval(10000L, sendSensorDataWithSMA); // Send sensor data with SMA every 10 seconds
    timer.setInterval(3000L, displaySensorData); // Update LCD display every 3 seconds
    // No need to set interval for pollution_alert, it will be triggered by BLYNK_WRITE(V20)

    // Run Blynk
    Blynk.run();
}

void loop() {
    timer.run(); // Run Blynk timer routines
}
