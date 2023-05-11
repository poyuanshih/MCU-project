---
layout: post
title: ESP32 IoT webserver & client Homework
author: [Poyuan Shih]
category: [Lecture]
tags: [jekyll, ai]
---
### IoT webserver & client初始操作
## 將作為接受器的esp32寫入以下程式
## esp32接受程式

```
//
// ESP32 Webserver to receive data from Webclients
// To use a web browser to open IP address of this webserver 
//
#include <WiFi.h> 
#include <WebServer.h>

const char* ssid     = "Test";
const char* password = "tonyikci";

WebServer server(80);

const String HTTP_PAGE_HEAD  = "<!DOCTYPE html><html lang=\"en\"><head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1, user-scalable=no\"/><title>{v}</title>";
const String HTTP_PAGE_STYLE = "<style>.c{text-align: center;} div,input{padding:5px;font-size:1em;}  input{width:90%;}  body{text-align: center;font-family:verdana;} button{border:0;border-radius:0.6rem;background-color:#1fb3ec;color:#fdd;line-height:2.4rem;font-size:1.2rem;width:100%;} .q{float: right;width: 64px;text-align: right;} .button2 {background-color: #008CBA;} .button3 {background-color: #f44336;} .button4 {background-color: #e7e7e7; color: black;} .button5 {background-color: #555555;} .button6 {background-color: #4CAF50;} </style>";
const String HTTP_PAGE_SCRIPT= "<script>function c(l){document.getElementById('s').value=l.innerText||l.textContent;document.getElementById('p').focus();}</script>";
const String HTTP_PAGE_BODY  = "</head><body><div style='text-align:left;display:inline-block;min-width:260px;'>";
const String HTTP_WEBPAGE = HTTP_PAGE_HEAD + HTTP_PAGE_STYLE + HTTP_PAGE_SCRIPT + HTTP_PAGE_BODY;

const String HTTP_PAGE_END = "</div></body></html>";

// DHT22 sensor data
String HTU21D_name0 = "Temperature";
String HTU21D_name1 = "Humidity";
String HTU21D_value0= "0";
String HTU21D_value1= "0";
/* HTU21DF sensor data
String htu21_name0 = "Temperature";
String htu21_name1 = "Humidity";
String htu21_value0= "0 ";
String htu21_value1= "0 ";
*/
/* PM5003 sensor data
String pm_name0 = "PM1.0";
String pm_name1 = "PM2.5";
String pm_name2 = "PM10.0";
String pm_value0= "0 ug/m3";
String pm_value1= "0 ug/m3";
String pm_value2= "0 ug/m3";
*/
void handleRoot() {
  // Display Sensor Status
  String s  = HTTP_WEBPAGE;
         s += "<table border=\"1\"";
         s += "<tr><th align='center'>HTU21D Sensor</th><th align='cener'>value</th></tr>";
         s += "<tr><td align='center'>"+HTU21D_name0+"</td><td align='center'>"+HTU21D_value0+"</td></tr>";
         s += "<tr><td align='center'>"+HTU21D_name1+"</td><td align='center'>"+HTU21D_value1+"</td></tr>";
         /*
         s += "<tr><th align='center'>HTU21 Sensor</th><th align='cener'>value</th></tr>";       
         s += "<tr><td align='center'>"+htu21_name0+"</td><td align='center'>"+htu21_value0+"</td></tr>";
         s += "<tr><td align='center'>"+htu21_name1+"</td><td align='center'>"+htu21_value1+"</td></tr>";
         s += "<tr><th align='center'>PM5003 Sensor</th><th align='cener'>value</th></tr>";
         s += "<tr><td align='center'>"+pm_name0+"</td><td align='center'>"+pm_value0+"</td></tr>";
         s += "<tr><td align='center'>"+pm_name1+"</td><td align='center'>"+pm_value1+"</td></tr>";
         s += "<tr><td align='center'>"+pm_name2+"</td><td align='center'>"+pm_value2+"</td></tr>";   
         */            
         s += "</tr></table>";
         s += HTTP_PAGE_END;
         
  server.send(200, "text/html", s);
}

// http://192.168.1.12/dht22?T=28&H=50 emulate data from Webclient_DHT22
//(you can open a browser to test it, too)
void HTU21D() {
  String message = "Number of args received:";
  message += server.args();                   //Get number of parameters
  message += "\n";                            //Add a new line

  for (int i = 0; i < server.args(); i++) {
    message += "Arg "+(String)i + " –> "; //Include the current iteration value
    message += server.argName(i) + ": ";      //Get the name of the parameter
    message += server.arg(i) + "\n";          //Get the value of the parameter
  }
  Serial.print(message);

  HTU21D_value0=server.arg(0);
  HTU21D_value1=server.arg(1);
  
  String s  = HTTP_WEBPAGE;
         s += "<table border=\"1\"";
         s += "<tr><th align='center'>DHT22 Sensor</th><th align='cener'>value</th></tr>";
         s += "<tr><td align='center'>"+HTU21D_name0+"</td><td align='center'>"+HTU21D_value0+"</td></tr>";
         s += "<tr><td align='center'>"+HTU21D_name1+"</td><td align='center'>"+HTU21D_value1+"</td></tr>";         
         s += "</tr></table>";
         s += HTTP_PAGE_END; 
   
  server.send(200, "text/html", s);
}

// http://192.168.1.12/htu21?T=28&H=50 emulate data from Webclient_HTU21
//(you can open a browser to test it, too)
void setup() {
  Serial.begin(115200);
  
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");  
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/HTU21D", HTU21D);

  Serial.println("HTTP server started");
  server.begin();  
}

void loop() {
  server.handleClient();
}
```
## 將作為發射器的esp32連接HTU21D寫入以下程式
## ESP32傳送訊號程式
```
// Webclient to read HTU21DF and send data to Webserver
#include <WiFi.h>
#include <WiFiClient.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_HTU21DF.h>

// Connect Vin to 3-5VDC
// Connect GND to ground
// Connect SCL to I2C clock pin (A5 on UNO, D1 on NodeMCU)
// Connect SDA to I2C data pin (A4 on UNO, D2 on NodeMCU)

Adafruit_HTU21DF htu = Adafruit_HTU21DF();

const char* ssid     = "Test";
const char* password = "tonyikci";
String      webserverIP = "http://192.168.109.244"; // Your Webserver IP address

void setup() {
  Serial.begin(115200);
  Serial.println("HTU21D-F test");  
  if (!htu.begin()) {
    Serial.println("Couldn't find HTU21DF sensor!");
    while (1);
  }
  
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");  
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  delay(5000);

  float temp  = htu.readTemperature();
  float humid = htu.readHumidity();
  Serial.print(temp);
  Serial.print(" ");
  Serial.print(humid);
  Serial.println();
  
  String url = webserverIP + "/HTU21D?";
  url += "T=";
  url += String(temp);
  url += "&H=";
  url += String(humid);
  
  if(WiFi.status()== WL_CONNECTED){   //Check WiFi connection status

     WiFiClient client;

     HTTPClient http;    //Declare object of class HTTPClient
 
     http.begin(client, url);    //Specify request destination
     http.addHeader("Content-Type", "text/plain");  //Specify content-type header
 
     int httpCode = http.POST("Message from ESP32");   //Send the request
     String payload = http.getString();                //Get the response payload
 
     Serial.println(httpCode);   //Print HTTP return code
     Serial.println(payload);    //Print request response payload

     http.end();  //Close connection
 
   }else{
      Serial.println("Error in WiFi connection");   
   }
  
  Serial.println();
  Serial.println("closing connection. going to sleep...");
  // go to deepsleep for 1 minutes
  //system_deep_sleep_set_option(0);
  //system_deep_sleep(1 * 60 * 1000000);
  delay(1*60*100);
}
```
## 將發射器連接HTU21D
![](https://s4.aconvert.com/convert/p3r68-cdx67/a5rmv-r41r0.png)
## 成果展示
## 網頁：
## 初始狀態
![](https://s4.aconvert.com/convert/p3r68-cdx67/akzye-45dyc.png)
## 訊息輸入後狀態
![](https://s4.aconvert.com/convert/p3r68-cdx67/apcas-hf908.png)
## 序列埠狀態對比
![](https://s4.aconvert.com/convert/p3r68-cdx67/awql7-j38fc.png)
