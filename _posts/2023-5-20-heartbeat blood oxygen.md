---
layout: post
title: EBO作業
author: [poyuanshih]
category: [Lecture]
tags: [IMU]
---
## 心跳血氧感測器
### 系統方塊圖
![](https://github.com/poyuanshih/MCU-project/blob/main/images/img4.png?raw=true)
### Arduino程式

```
// MAX30102 + 128x32 OLED(I2C)
#include <Adafruit_GFX.h>        //OLED libraries
#include <Adafruit_SSD1306.h>
#include <Wire.h>
#include "MAX30105.h"           //MAX3010x library
#include "heartRate.h"          //Heart rate calculating algorithm
#include "ESP32Servo.h"
MAX30105 particleSensor;
int Tonepin = 4;
//計算心跳用變數
const byte RATE_SIZE = 10; //多少平均數量
byte rates[RATE_SIZE]; //心跳陣列
byte rateSpot = 0;
long lastBeat = 0; //Time at which the last beat occurred
float beatsPerMinute;
int beatAvg;

//計算血氧用變數
double avered = 0;
double aveir = 0;
double sumirrms = 0;
double sumredrms = 0;

double SpO2 = 0;
double ESpO2 = 90.0;//初始值
double FSpO2 = 0.7; //filter factor for estimated SpO2
double frate = 0.95; //low pass filter for IR/red LED value to eliminate AC component
int i = 0;
int Num = 30;//取樣100次才計算1次
#define FINGER_ON 7000 //紅外線最小量（判斷手指有沒有上）
#define MINIMUM_SPO2 90.0//血氧最小量

//OLED設定
#define SCREEN_WIDTH 128 //OLED寬度
#define SCREEN_HEIGHT 64 //OLED高度
#define OLED_RESET    -1 //Reset pin
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET); //Declaring the display name (display)

//心跳小圖
static const unsigned char PROGMEM logo2_bmp[] =
{ 0x03, 0xC0, 0xF0, 0x06, 0x71, 0x8C, 0x0C, 0x1B, 0x06, 0x18, 0x0E, 0x02, 0x10, 0x0C, 0x03, 0x10,              //Logo2 and Logo3 are two bmp pictures that display on the OLED if called
  0x04, 0x01, 0x10, 0x04, 0x01, 0x10, 0x40, 0x01, 0x10, 0x40, 0x01, 0x10, 0xC0, 0x03, 0x08, 0x88,
  0x02, 0x08, 0xB8, 0x04, 0xFF, 0x37, 0x08, 0x01, 0x30, 0x18, 0x01, 0x90, 0x30, 0x00, 0xC0, 0x60,
  0x00, 0x60, 0xC0, 0x00, 0x31, 0x80, 0x00, 0x1B, 0x00, 0x00, 0x0E, 0x00, 0x00, 0x04, 0x00,
};
//心跳大圖
static const unsigned char PROGMEM logo3_bmp[] =
{ 0x01, 0xF0, 0x0F, 0x80, 0x06, 0x1C, 0x38, 0x60, 0x18, 0x06, 0x60, 0x18, 0x10, 0x01, 0x80, 0x08,
  0x20, 0x01, 0x80, 0x04, 0x40, 0x00, 0x00, 0x02, 0x40, 0x00, 0x00, 0x02, 0xC0, 0x00, 0x08, 0x03,
  0x80, 0x00, 0x08, 0x01, 0x80, 0x00, 0x18, 0x01, 0x80, 0x00, 0x1C, 0x01, 0x80, 0x00, 0x14, 0x00,
  0x80, 0x00, 0x14, 0x00, 0x80, 0x00, 0x14, 0x00, 0x40, 0x10, 0x12, 0x00, 0x40, 0x10, 0x12, 0x00,
  0x7E, 0x1F, 0x23, 0xFE, 0x03, 0x31, 0xA0, 0x04, 0x01, 0xA0, 0xA0, 0x0C, 0x00, 0xA0, 0xA0, 0x08,
  0x00, 0x60, 0xE0, 0x10, 0x00, 0x20, 0x60, 0x20, 0x06, 0x00, 0x40, 0x60, 0x03, 0x00, 0x40, 0xC0,
  0x01, 0x80, 0x01, 0x80, 0x00, 0xC0, 0x03, 0x00, 0x00, 0x60, 0x06, 0x00, 0x00, 0x30, 0x0C, 0x00,
  0x00, 0x08, 0x10, 0x00, 0x00, 0x06, 0x60, 0x00, 0x00, 0x03, 0xC0, 0x00, 0x00, 0x01, 0x80, 0x00
};
//氧氣圖示
static const unsigned char PROGMEM O2_bmp[] = {
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xf0, 0x00, 0x3f, 0xc3, 0xf8, 0x00, 0xff, 0xf3, 0xfc,
  0x03, 0xff, 0xff, 0xfe, 0x07, 0xff, 0xff, 0xfe, 0x0f, 0xff, 0xff, 0xfe, 0x0f, 0xff, 0xff, 0x7e,
  0x1f, 0x80, 0xff, 0xfc, 0x1f, 0x00, 0x7f, 0xb8, 0x3e, 0x3e, 0x3f, 0xb0, 0x3e, 0x3f, 0x3f, 0xc0,
  0x3e, 0x3f, 0x1f, 0xc0, 0x3e, 0x3f, 0x1f, 0xc0, 0x3e, 0x3f, 0x1f, 0xc0, 0x3e, 0x3e, 0x2f, 0xc0,
  0x3e, 0x3f, 0x0f, 0x80, 0x1f, 0x1c, 0x2f, 0x80, 0x1f, 0x80, 0xcf, 0x80, 0x1f, 0xe3, 0x9f, 0x00,
  0x0f, 0xff, 0x3f, 0x00, 0x07, 0xfe, 0xfe, 0x00, 0x0b, 0xfe, 0x0c, 0x00, 0x1d, 0xff, 0xf8, 0x00,
  0x1e, 0xff, 0xe0, 0x00, 0x1f, 0xff, 0x00, 0x00, 0x1f, 0xf0, 0x00, 0x00, 0x1f, 0xe0, 0x00, 0x00,
  0x0f, 0xe0, 0x00, 0x00, 0x07, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
};

void setup() {
  Serial.begin(115200);
  Serial.println("System Start");
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C); //Start the OLED display
  display.display();
  delay(3000);
  //檢查
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) //Use default I2C port, 400kHz speed
  {
    Serial.println("找不到MAX30102");
    while (1);
  }
  byte ledBrightness = 0x7F; //亮度Options: 0=Off to 255=50mA
  byte sampleAverage = 4; //Options: 1, 2, 4, 8, 16, 32
  byte ledMode = 2; //Options: 1 = Red only(心跳), 2 = Red + IR(血氧)
  //Options: 1 = IR only, 2 = Red + IR on MH-ET LIVE MAX30102 board
  int sampleRate = 800; //Options: 50, 100, 200, 400, 800, 1000, 1600, 3200
  int pulseWidth = 215; //Options: 69, 118, 215, 411
  int adcRange = 16384; //Options: 2048, 4096, 8192, 16384
  // Set up the wanted parameters
  particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange); //Configure sensor with these settings
  particleSensor.enableDIETEMPRDY();

  particleSensor.setPulseAmplitudeRed(0x0A); //Turn Red LED to low to indicate sensor is running
  particleSensor.setPulseAmplitudeGreen(0); //Turn off Green LED
}

void loop() {
  long irValue = particleSensor.getIR();    //Reading the IR value it will permit us to know if there's a finger on the sensor or not
  //是否有放手指
  if (irValue > FINGER_ON ) {
    display.clearDisplay();//清除螢幕
    display.drawBitmap(5, 5, logo2_bmp, 24, 21, WHITE);//顯示小的心跳圖示
    display.setTextSize(2);//設定文字大小
    display.setTextColor(WHITE);//文字顏色
    display.setCursor(42, 10);//設定游標位置
    display.print(beatAvg); display.println(" BPM");//顯示心跳數值
    display.drawBitmap(0, 35, O2_bmp, 32, 32, WHITE);//顯示氧氣圖示
    display.setCursor(42, 40);//設定游標位置
    //顯示血氧數值
    if (beatAvg > 30) display.print(String(ESpO2) + "%");
    else display.print("---- %" );
    display.display();//顯示螢幕
    //是否有心跳
    if (checkForBeat(irValue) == true) {
      display.clearDisplay();//清除螢幕
      display.drawBitmap(0, 0, logo3_bmp, 32, 32, WHITE);//顯示大的心跳圖示
      display.setTextSize(2);//設定文字大小
      display.setTextColor(WHITE);//文字顏色
      display.setCursor(42, 10);//設定游標位置
      display.print(beatAvg); display.println(" BPM");//顯示心跳數值
      display.drawBitmap(0, 35, O2_bmp, 32, 32, WHITE);//顯示氧氣圖示
      display.setCursor(42, 40);//設定游標位置
      //顯示血氧數值
      if (beatAvg > 30) display.print(String(ESpO2) + "%");
      else display.print("---- %" );
      display.display();//顯示螢幕
      tone(Tonepin, 1000);//發出聲音
      delay(10);
      noTone(Tonepin);//停止聲音
      Serial.print("beatAvg="); Serial.println(beatAvg);//將心跳顯示到序列
      long delta = millis() - lastBeat;//計算心跳差
      lastBeat = millis();
      beatsPerMinute = 60 / (delta / 1000.0);//計算平均心跳
      if (beatsPerMinute < 255 && beatsPerMinute > 20) {
        //心跳必須再20-255之間
        rates[rateSpot++] = (byte)beatsPerMinute; //儲存心跳數值陣列
        rateSpot %= RATE_SIZE;
        beatAvg = 0;//計算平均值
        for (byte x = 0 ; x < RATE_SIZE ; x++) beatAvg += rates[x];
        beatAvg /= RATE_SIZE;
      }
    }

    //計算血氧
    uint32_t ir, red ;
    double fred, fir;
    particleSensor.check(); //Check the sensor, read up to 3 samples
    if (particleSensor.available()) {
      i++;
      red = particleSensor.getFIFOIR(); //讀取紅光
      ir = particleSensor.getFIFORed(); //讀取紅外線
      //Serial.println("red=" + String(red) + ",IR=" + String(ir) + ",i=" + String(i));
      fred = (double)red;//轉double
      fir = (double)ir;//轉double
      avered = avered * frate + (double)red * (1.0 - frate);//average red level by low pass filter
      aveir = aveir * frate + (double)ir * (1.0 - frate); //average IR level by low pass filter
      sumredrms += (fred - avered) * (fred - avered); //square sum of alternate component of red level
      sumirrms += (fir - aveir) * (fir - aveir);//square sum of alternate component of IR level
      if ((i % Num) == 0) {
        double R = (sqrt(sumredrms) / avered) / (sqrt(sumirrms) / aveir);
        SpO2 = -23.3 * (R - 0.4) + 100;
        ESpO2 = FSpO2 * ESpO2 + (1.0 - FSpO2) * SpO2;//low pass filter
        if (ESpO2 <= MINIMUM_SPO2) ESpO2 = MINIMUM_SPO2; //indicator for finger detached
        if (ESpO2 > 100) ESpO2 = 99.9;
        Serial.print("Oxygen % = "); Serial.println(ESpO2);
        sumredrms = 0.0; sumirrms = 0.0; SpO2 = 0;
        i = 0;
      }
      particleSensor.nextSample(); //We're finished with this sample so move to next sample
    }

  } else {
    //清除心跳數據
    for (byte rx = 0 ; rx < RATE_SIZE ; rx++) rates[rx] = 0;
    beatAvg = 0; rateSpot = 0; lastBeat = 0;
    //清除血氧數據
    avered = 0; aveir = 0; sumirrms = 0; sumredrms = 0;
    SpO2 = 0; ESpO2 = 90.0;
    //顯示Finger Please
    display.clearDisplay();
    display.setTextSize(2);
    display.setTextColor(WHITE);
    display.setCursor(30, 5);
    display.println("Finger");
    display.setCursor(30, 35);
    display.println("Please");
    display.display();
    noTone(Tonepin);
  }
}
```

**將以上程式燒入esp32**
## 成果展示
![](https://github.com/poyuanshih/MCU-project/blob/main/images/img9.png?raw=true)<br>
