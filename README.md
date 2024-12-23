#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include <Firebase_ESP_Client.h>

// Provide the token generation process info
#include "addons/TokenHelper.h"
// Provide the RTDB payload printing info and other helper functions
#include "addons/RTDBHelper.h"

// Insert your network credentials
#define WIFI_SSID "MOHAN"
#define WIFI_PASSWORD "123456789"

// Insert Firebase project API Key
#define API_KEY "AIzaSyCTpCWhdAcCEiSlV5FIOi4cYahIBjRDl_8"

// Insert RTDB URL
#define DATABASE_URL "https://fuel-level-monitoring-sy-ec88a-default-rtdb.asia-southeast1.firebasedatabase.app/"

// Define Firebase objects
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

bool signupOK = false;

// Pin and Threshold Configurations
const int analogPin = 13;
const int minValue = 2750;           // Minimum analog value
const int maxValue = 280;            // Maximum analog value
const int lowFuelThreshold = 10;     // Low fuel threshold (percentage)
const int suddenDropThreshold = 20;  // Sudden drop threshold (percentage)

// Initialize the LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

float lastPercentage = 0;  // Store the last percentage for sudden drop detection

void setup() {
  Serial.begin(115200);

  // Connect to Wi-Fi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(300);
  }
  Serial.println();
  Serial.print("Connected with IP: ");
  Serial.println(WiFi.localIP());
  Serial.println();

  // Initialize the LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("   Fuel Level   ");
  lcd.setCursor(0, 1);
  lcd.print("   Indicator    ");
  delay(2000);
  lcd.clear();

  // Firebase configuration
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;

  // Sign up for Firebase
  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("Sign up successful");
    signupOK = true;
  } else {
    Serial.printf("Sign up error: %s\n", config.signer.signupError.message.c_str());
  }

  // Assign the callback function for token status
  config.token_status_callback = tokenStatusCallback;  // See addons/TokenHelper.h

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}

void loop() {
  int analogValue = analogRead(analogPin);

  // Convert analog value to percentage
  float percentage = ((float)(analogValue - minValue) / (maxValue - minValue)) * 100;
  percentage = constrain(percentage, 0, 100);

  // Print to Serial
  Serial.print("Analog Value: ");
  Serial.print(analogValue);
  Serial.print(" | Percentage: ");
  Serial.print(percentage);
  Serial.println("%");

  // Display on LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Fuel Level :");
  lcd.setCursor(5, 1);
  lcd.print(percentage);
  lcd.print("%");

  // Update Firebase
  if (Firebase.RTDB.setFloat(&fbdo, "/FuelLevel", percentage)) {
    Serial.println("Fuel Level updated");
  } else {
    Serial.println("Failed to update Fuel Level");
    Serial.println(fbdo.errorReason());
  }

  // Low fuel alert
  if (percentage < lowFuelThreshold) {
    Serial.println("Low Fuel Alert!");
    lcd.setCursor(0, 1);
    lcd.print("Low Fuel Alert!");
    Firebase.RTDB.setString(&fbdo, "/Alert", "Low Fuel Alert");
  }

  // Sudden drop alert
  if (abs(percentage - lastPercentage) > suddenDropThreshold) {
    Serial.println("Sudden Drop Alert!");
    lcd.setCursor(0, 1);
    lcd.print("Sudden Drop!");
    Firebase.RTDB.setString(&fbdo, "/Alert", "Sudden Drop Alert");
  }

  lastPercentage = percentage;  // Update last percentage
  delay(500);
}
