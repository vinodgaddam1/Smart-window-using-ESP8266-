#include <ESP8266WiFi.h>
#include <FirebaseESP8266.h>
#include <DHT.h>
#include <Servo.h>

// Wi-Fi Credentials
#define WIFI_SSID "Vamsi"
#define WIFI_PASSWORD "vamsi123"

// Firebase Credentials
#define FIREBASE_HOST "smartwindow-21700-default-rtdb.firebaseio.com"
#define FIREBASE_AUTH "WPgGaAcUk01bY7VzNpm7FHlm2jszBavYWaKFfyhj"

// Sensor & Actuator Pins
#define DHTPIN D4
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

#define MQ135_PIN A0
#define RAIN_SENSOR D5
#define LDR_SENSOR D0
#define SERVO_PIN D6
#define BUZZER D7

Servo windowServo;

// Threshold Values
#define TEMP_THRESHOLD 10
#define LIGHT_THRESHOLD 200
#define RAIN_THRESHOLD 1
#define AIR_QUALITY_THRESHOLD 100

// Firebase Configuration
FirebaseData firebaseData;
FirebaseAuth auth;
FirebaseConfig config;

void connectToWiFi() {
    Serial.print("Connecting to WiFi");
    WiFi.begin("Vamsi", "vamsi123");
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("\nWiFi Connected!");
}

void publishDataToFirebase(float temperature, int lightValue, int airQuality, int rainValue) {
    // Prepare JSON payload
    FirebaseJson json;
    json.set("temperature", temperature);
    json.set("light", lightValue);
    json.set("air_quality", airQuality);
    json.set("rain", rainValue);

    // Send data to Firebase
    if (Firebase.pushJSON(firebaseData, "/smart-window/data", json)) {
        Serial.println("Data published to Firebase.");
    } else {
        Serial.print("Failed to publish data: ");
        Serial.println(firebaseData.errorReason());
    }
}

void setup() {
    Serial.begin(115200);

    connectToWiFi();

    // Set up Firebase configuration
    config.host = "smartwindow-21700-default-rtdb.firebaseio.com";
    config.signer.tokens.legacy_token ="WPgGaAcUk01bY7VzNpm7FHlm2jszBavYWaKFfyhj";

    // Initialize Firebase
    Firebase.begin(&config, &auth);
    Firebase.reconnectWiFi(true);

    dht.begin();
    windowServo.attach(SERVO_PIN);
    pinMode(RAIN_SENSOR, INPUT);
    pinMode(BUZZER, OUTPUT);

    Serial.println("Smart Window System Initialized.");
}

void loop() {
    // Read Sensor Data
    float temperature = dht.readTemperature();
    int lightValue = analogRead(LDR_SENSOR);
    int airQuality = analogRead(MQ135_PIN);
    int rainValue = analogRead(RAIN_SENSOR);

    // Publish data to Firebase
    publishDataToFirebase(temperature, lightValue, airQuality, rainValue);

    // Predictive Mode - Auto Open/Close based on Environment
    if (temperature > TEMP_THRESHOLD || lightValue > LIGHT_THRESHOLD ||airQuality > AIR_QUALITY_THRESHOLD) 
    {
         if ( rainValue > RAIN_THRESHOLD)
         {
          digitalWrite(BUZZER, HIGH);
          windowServo.write(90);
          }
    else
    {
        digitalWrite(BUZZER, HIGH);
        windowServo.write(0); // Open window
        Serial.println("Window opened due to environmental conditions.");
        }
    }
    delay(20000); // Update every 20 seconds
    }
