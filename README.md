<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# **Complete Step-by-Step Guide: GPS-Based School Bus Tracking System**

## **Phase 1: Software Setup \& ESP32 Testing**

### **Step 1: Install Arduino IDE**

1. **Download Arduino IDE** from https://www.arduino.cc/en/software
2. **Install** the downloaded file (choose latest version)
3. **Open Arduino IDE** after installation

### **Step 2: Add ESP32 Board Support**

1. **Open File → Preferences** in Arduino IDE
2. **In "Additional Board Manager URLs"** add:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

3. **Click OK**
4. **Go to Tools → Board → Boards Manager**
5. **Search for "ESP32"** and install **"ESP32 by Espressif Systems"**
6. **Wait for installation** to complete (may take 5-10 minutes)

### **Step 3: Test ESP32 Board**

1. **Connect ESP32** to your laptop using USB cable
2. **Go to Tools → Board** and select **"ESP32 Dev Module"**
3. **Go to Tools → Port** and select the COM port (e.g., COM3, COM4)
4. **Upload this test code**:
```cpp
void setup() {
  Serial.begin(115200);
  Serial.println("ESP32 Test - Hello World!");
}

void loop() {
  Serial.println("ESP32 is working!");
  delay(2000);
}
```

5. **Click Upload button** (→ arrow icon)
6. **Open Serial Monitor** (Tools → Serial Monitor)
7. **Set baud rate to 115200**
8. **You should see**: "ESP32 is working!" every 2 seconds

**✅ If you see this output, your ESP32 is working correctly!**

***

## **Phase 2: Hardware Connections**

### **Step 4: Gather Components**

- ESP32 Development Board
- Neo-6M GPS Module
- Breadboard
- Jumper wires
- 2 LEDs (Red and Green)
- 2 Resistors (220Ω or 330Ω)


### **Step 5: Circuit Connections**

**Neo-6M GPS to ESP32:**

```
GPS Module    →    ESP32
VCC           →    3.3V
GND           →    GND
TX            →    GPIO 4 (RX)
RX            →    GPIO 2 (TX)
```

**LEDs to ESP32:**

```
Green LED + Resistor → GPIO 25
Red LED + Resistor   → GPIO 26
```


### **Step 6: Test GPS Module**

Upload this code to test GPS:

```cpp
#include <HardwareSerial.h>

HardwareSerial gpsSerial(1);

void setup() {
  Serial.begin(115200);
  gpsSerial.begin(9600, SERIAL_8N1, 4, 2);
  Serial.println("GPS Test Started");
}

void loop() {
  if (gpsSerial.available()) {
    String gpsData = gpsSerial.readString();
    Serial.print("GPS Data: ");
    Serial.println(gpsData);
  }
  delay(1000);
}
```

**Expected Output:** You should see GPS NMEA sentences like:

```
$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
```


***

## **Phase 3: WiFi Setup \& API Registration**

### **Step 7: Test WiFi Connection**

```cpp
#include <WiFi.h>

const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";

void setup() {
  Serial.begin(115200);
  
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  
  Serial.println("");
  Serial.println("WiFi connected!");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  Serial.println("WiFi Status: Connected");
  delay(5000);
}
```

**✅ Replace "YOUR_WIFI_NAME" and "YOUR_WIFI_PASSWORD" with your actual WiFi credentials**

### **Step 8: Register for Circuit Digest API**

1. **Go to** https://circuitdigest.cloud/
2. **Click "Register"** (create free account)
3. **Verify your email**
4. **Log in** to your account
5. **Go to "My Account"** → copy your **API Key**
6. **Keep this API key safe** - you'll need it in the code

***

## **Phase 4: Complete GPS Tracker Code**

### **Step 9: Install Required Libraries**

1. **Go to Tools → Manage Libraries**
2. **Search and install**:
    - **TinyGPS++** by Mikal Hart
    - **ArduinoJson** by Benoit Blanchon

### **Step 10: Complete GPS Tracker Code**

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <TinyGPS++.h>
#include <HardwareSerial.h>
#include <ArduinoJson.h>
#include <SPIFFS.h>

// WiFi credentials
const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";

// Circuit Digest API
const char* apiKey = "YOUR_API_KEY_HERE";
const char* apiUrl = "https://circuitdigest.cloud/api/geolinker/store";

// GPS Setup
TinyGPSPlus gps;
HardwareSerial gpsSerial(1);

// LED Pins
#define GREEN_LED 25
#define RED_LED 26

void setup() {
  Serial.begin(115200);
  
  // Initialize LEDs
  pinMode(GREEN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  // Start GPS
  gpsSerial.begin(9600, SERIAL_8N1, 4, 2);
  
  // Initialize SPIFFS
  if (!SPIFFS.begin(true)) {
    Serial.println("SPIFFS initialization failed");
    return;
  }
  
  // Connect to WiFi
  connectToWiFi();
  
  Serial.println("GPS Tracker Started!");
}

void loop() {
  // Read GPS data
  while (gpsSerial.available() > 0) {
    if (gps.encode(gpsSerial.read())) {
      if (gps.location.isValid()) {
        digitalWrite(RED_LED, HIGH);
        
        double latitude = gps.location.lat();
        double longitude = gps.location.lng();
        
        Serial.print("Latitude: ");
        Serial.println(latitude, 6);
        Serial.print("Longitude: ");
        Serial.println(longitude, 6);
        
        // Send to API
        sendLocationData(latitude, longitude);
        
      } else {
        digitalWrite(RED_LED, LOW);
        Serial.println("Invalid GPS data, skipping upload");
      }
    }
  }
  
  delay(10000); // Send data every 10 seconds
}

void connectToWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
    digitalWrite(GREEN_LED, !digitalRead(GREEN_LED)); // Blink while connecting
  }
  
  digitalWrite(GREEN_LED, HIGH); // Solid green when connected
  Serial.println("");
  Serial.println("WiFi connected!");
}

void sendLocationData(double lat, double lon) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(apiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Create JSON payload
    StaticJsonDocument<200> jsonDoc;
    jsonDoc["api_key"] = apiKey;
    jsonDoc["latitude"] = lat;
    jsonDoc["longitude"] = lon;
    
    String jsonString;
    serializeJson(jsonDoc, jsonString);
    
    int httpResponseCode = http.POST(jsonString);
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.println("Data uploaded successfully!");
      Serial.println("Response code: " + String(httpResponseCode));
    } else {
      Serial.println("Error uploading data");
    }
    
    http.end();
  } else {
    Serial.println("WiFi not connected, saving data offline");
    // Here you can add code to save data to SPIFFS for later upload
  }
}
```

**✅ Remember to replace:**

- `YOUR_WIFI_NAME` with your WiFi network name
- `YOUR_WIFI_PASSWORD` with your WiFi password
- `YOUR_API_KEY_HERE` with your Circuit Digest API key

***

## **Phase 5: Testing \& Viewing Results**

### **Step 11: Upload and Test**

1. **Upload the complete code** to ESP32
2. **Open Serial Monitor** (115200 baud)
3. **Take ESP32 outside** or near a window for GPS signal
4. **Wait 1-2 minutes** for GPS to get satellite fix
5. **You should see**:

```
WiFi connected!
Latitude: 12.345678
Longitude: 77.123456
Data uploaded successfully!
Response code: 201
```


### **Step 12: View Live Tracking**

1. **Go to** https://circuitdigest.cloud/
2. **Log in** with your account
3. **Click "View Map"** next to GeoLinker API
4. **You should see** your location plotted on Google Maps
5. **Blue dots** show your GPS tracker's path
6. **Click on dots** to see timestamp

***

## **Phase 6: Add School Bus Features**

### **Step 13: Add SMS Notifications (Optional)**

If you have SIM800L GSM module:

```cpp
// Add this to your main code
#include <SoftwareSerial.h>

SoftwareSerial gsm(16, 17);

void sendSMS(String message, String phoneNumber) {
  gsm.println("AT");
  delay(1000);
  gsm.println("AT+CMGF=1");
  delay(1000);
  gsm.print("AT+CMGS=\"");
  gsm.print(phoneNumber);
  gsm.println("\"");
  delay(1000);
  gsm.print(message);
  delay(100);
  gsm.println((char)26);
  delay(1000);
}
```


### **Step 14: Add Geo-fencing**

```cpp
// Add these variables at the top
double schoolLat = 12.345678;  // School coordinates
double schoolLon = 77.123456;
double homeLat = 12.987654;    // Home coordinates  
double homeLon = 77.654321;

// Add this function
double calculateDistance(double lat1, double lon1, double lat2, double lon2) {
  // Simple distance calculation
  double dLat = lat2 - lat1;
  double dLon = lon2 - lon1;
  return sqrt(dLat*dLat + dLon*dLon) * 111000; // Rough conversion to meters
}

// In your main loop, add:
double distanceToSchool = calculateDistance(latitude, longitude, schoolLat, schoolLon);
if (distanceToSchool < 100) { // Within 100 meters of school
  Serial.println("Bus reached school!");
  // sendSMS("Bus has reached school", "+91XXXXXXXXXX");
}
```


***

## **Phase 7: Troubleshooting**

### **Common Issues:**

**ESP32 not detected:**

- Try different USB cable
- Hold BOOT button while uploading
- Check if drivers are installed

**No GPS signal:**

- Take device outside
- Wait 2-5 minutes for satellite lock
- Check GPS module connections

**WiFi not connecting:**

- Verify SSID and password
- Check WiFi signal strength
- Try different WiFi network

**API not working:**

- Verify API key is correct
- Check internet connection
- Ensure account is activated

***

## **Final Demo Preparation**

### **What to Show:**

1. **Live tracking** on Circuit Digest map
2. **Serial Monitor** showing GPS coordinates
3. **LED indicators** (Green = WiFi, Red = GPS)
4. **Automatic updates** every 10 seconds
5. **Path visualization** on map

### **What to Explain:**

1. **Hardware setup** and connections
2. **Code structure** and key functions
3. **API integration** for cloud storage
4. **Real-world applications** for school buses
5. **Future enhancements** (SMS, mobile app)

**🎉 You now have a complete, working GPS tracking system that automatically updates location data to the cloud every 10 seconds!**

