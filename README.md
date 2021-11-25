# IOT-and-cloud-based-Weather-Monitoring-System
                  The main objective of this system is to detect, record and display various weather parameters, this makes use of sensors for detecting and monitoring weather parameters and then collected information is sent to the cloud which can be accessed using the internet.

#include <DHT.h>    //library DHT11
#include <ESP8266WiFi.h>  //library esp
#include <Wire.h>   //library bmp180
#include <Adafruit_Sensor.h>
#include <Adafruit_BMP085_U.h>

Adafruit_BMP085_Unified bmp = Adafruit_BMP085_Unified(10085);


// replace with your channel’s thingspeak API key,
String apiKey = "D9SLFAR0VFZLO6YT";                         //fill in the api key from thingspeak
const char* ssid = "XXX";                                  //fill in your wifi name
const char* password = "YYY";                          //fill in your wifi password

const char* server = "api.thingspeak.com";
#define DHTPIN 0    // what pin we’re connected to

DHT dht(DHTPIN, DHT11);
WiFiClient client;

int sensorPin = A0;       // input for  rain sensor

int enable1 = 13;         // enable reading Rain sensor

int RainSensor = 0;     // variable to store the value coming from sensor Rain sensor
  
//--------------------------setup-------------------------
void setup() {

// declare the enable  as an OUTPUT:
pinMode(enable1, OUTPUT);

Serial.begin(115200);
delay(10);

dht.begin();

WiFi.begin(ssid, password);

while (WiFi.status() != WL_CONNECTED) {
delay(500);
Serial.print(".");
}
Serial.println("");
Serial.println("WiFi connected");

}


void loop() {
//--------------------------DHT11-------------------------

float h = dht.readHumidity();
float t = dht.readTemperature();

if (isnan(h) || isnan(t)) {
Serial.println("Failed to read from DHT sensor!!");
return;
}

Serial.print("Temperature:      ");
Serial.print(t);
Serial.print(" degrees Celcius ");
Serial.println();

Serial.print("Humidity:         ");
Serial.print(h);
Serial.print("%");
Serial.println();


//---dew point with the temperature and the humidity---

double gamma = log(h/100) + ((17.62*t) / (243.5+t));
double dp = 243.5*gamma / (17.62-gamma);

Serial.print("Dew point:        ");
Serial.print(dp);
Serial.print(" degrees Celcius ");
Serial.println();

//--------------------------BMP180------------------------

if(!bmp.begin()) {
Serial.print("Failed to read from BMP sensor!!");
while(1);
}

sensors_event_t event;
bmp.getEvent(&event);

Serial.print("Pressure:         ");
Serial.print(event.pressure);
Serial.println(" hPa");

float temperature;
bmp.getTemperature(&temperature);
Serial.print("Temperature:      ");
Serial.print(temperature);
Serial.println(" degrees Celcius ");

//---measure the altitude with the temperature and the air pressure---

float seaLevelPressure = 1015;
Serial.print("Altitude:         "); 
Serial.print(bmp.pressureToAltitude(seaLevelPressure,event.pressure)); 
Serial.println(" m");



//--------------------------Rain Sensor-------------------------

digitalWrite(enable1, HIGH); 

delay(100);
RainSensor = analogRead(sensorPin);
RainSensor = constrain(RainSensor, 150, 440); 
RainSensor = map(RainSensor, 150, 440, 1023, 0); 

Serial.print("Rain value:       ");
Serial.println(RainSensor);
Serial.println();
delay(100);

digitalWrite(enable1, LOW);

//--------------------------thingspeak-------------------------

if (client.connect(server,80)) { // "184.106.153.149" or api.thingspeak.com
String postStr = apiKey;
postStr +="&field1=";
postStr += String(t);
postStr +="&field2=";
postStr += String(h);
postStr +="&field3=";
postStr += String(dp);
postStr +="&field4=";
postStr += String(event.pressure);
postStr +="&field5=";
postStr += String(temperature);
postStr +="&field6=";
postStr += String(RainSensor);
postStr +="&field7=";
postStr += String(bmp.pressureToAltitude(seaLevelPressure,event.pressure));
postStr += "\r\n\r\n\r\n\r\n\r\n\r\n\r\n";

client.print("POST /update HTTP/1.1\n");
client.print("Host: api.thingspeak.com\n");
client.print("Connection: close\n");
client.print("X-THINGSPEAKAPIKEY: "+apiKey+"\n");
client.print("Content-Type: application/x-www-form-urlencoded\n");
client.print("Content-Length: ");
client.print(postStr.length());
client.print("\n\n\n\n\n\n\n\n");
client.print(postStr  );

}
client.stop();

delay(15000);   // thingspeak needs minimum 15 sec delay between updates
}
