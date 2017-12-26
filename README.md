// Group-4-Project
// The following contains the code necessary to theoretically automate the temperature reading on 3-hour intervals,
// sending message if it gets out of regulation status.
// Disclaimer: Much of the code are borrowed from Konstantin Dimitrov and Borya, as mentioned in the slides. Original codes are made in the
// integration of the two projects, as well as repeated temperature-reading and messages.

// Include necessary libraries
#include <OneWire.h> 
#include <DallasTemperature.h>
#include <ESP8266WiFi.h>
#include "Gsender.h"

// Declare global variables
#pragma region Globals
// WIFI network name
const char* ssid = "Wells International School";
// WIFI network password
const char* password = "wis85onnut";
// Connection to WiFi check
uint8_t connection_state = 0;
// If not connected wait time to try again
uint16_t reconnect_interval = 10000;
#pragma endregion Globals

// Data wire is plugged into pin 2 on the Arduino 
#define ONE_WIRE_BUS 2

// Setup a oneWire instance to communicate with any OneWire devices (not just Maxim/Dallas temperature ICs)
OneWire oneWire(ONE_WIRE_BUS); 

// Pass oneWire reference to Dallas Temperature 
DallasTemperature sensors(&oneWire);

// WiFi connection
uint8_t WiFiConnect(const char* nSSID = nullptr, const char* nPassword = nullptr)
{
    static uint16_t attempt = 0;
    Serial.print("Connecting to ");
    if(nSSID) {
        WiFi.begin(nSSID, nPassword);  
        Serial.println(nSSID);
    } else {
        WiFi.begin(ssid, password);
        Serial.println(ssid);
    }

    uint8_t i = 0;
    while(WiFi.status()!= WL_CONNECTED && i++ < 50)
    {
        delay(200);
        Serial.print(".");
    }
    ++attempt;
    Serial.println("");
    if(i == 51) {
        Serial.print("Connection: TIMEOUT on attempt: ");
        Serial.println(attempt);
        if(attempt % 2 == 0)
            Serial.println("Check if access point available or SSID and Password\r\n");
        return false;
    }
    Serial.println("Connection: ESTABLISHED");
    Serial.print("Got IP address: ");
    Serial.println(WiFi.localIP());
    return true;
}

// Ensuring establishment of WiFi connection
void Awaits()
{
uint32_t ts = millis();
while(!connection_state)
{
delay(50);
if(millis() > (ts + reconnect_interval) && !connection_state) {
           	connection_state = WiFiConnect();
           	ts = millis();
      	}
     }
}

void setup(void) 
{ 
// start serial port for temperature probe
Serial.begin(9600); 
// Start up the temp probe
sensors.begin();
// start serial port for WiFi module
Serial.begin(115200);
// Begin connection to WiFi
connection_state = WiFiConnect();
// Constant attempt to connect to WiFi
if(!connection_state){
Awaits();
}
Gsender *gsender = Gsender::Instance();
String subject = "Turtle Temp Monitor Starting Up";
// Send initial test message to confirm working
gsender->Subject(subject)->Send("wellsturtling@gmail.com", "Turtle Temp - Initiated");
// Reset subject line
String subject = “Turtle Temp Alert”;
} 

void loop(void)
{ 
// Get temperature reading
sensors.requestTemperatures();
// Get temperature reading in Celsius from first IC on wire
const float tempC = sensors.getTempCByIndex(0);
// If temperature goes out of regulation, send alert email
if (tempC < 23.9 || tempC > 32) {
	if (tempC < 23.9) {
	gsender->Subject(subject)->Send("boris.on@live.com", "Low Temp Alert: " + tempC);
} else {
	gsender->Subject(subject)->Send("boris.on@live.com", "High Temp Alert: " + tempC);
}
}
// Delay reading by 3 hrs
delay(10800000);
}
