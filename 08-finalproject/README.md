# Final Project

## inspiration
When I was exploring in urban nature, I focused my attention on insects like ants, and I felt attracted and inspired by their ability to track pheromones following a route based on the last ant. 

I wanted to make a device that had a similar mechanism. The aim is to generate interest among people for these ubiquitous insects found in both urban and natural environments.

## Description
This device is a head-worn apparatus with a biomimetic design resembling ant antenna. Its purpose is to emulate the navigation function among ants, which involves the exchange of pheromones to communicate path information. It utilizes an accelerometer to record the deviation angles and distances between path points, and employs a vibration motor to signal changes in direction. 
![picture description](./images/indoor1.jpg)
![picture description](./images/indoor2.jpg)

It has two modes, the first is track record. Touch the touch sensor to record the orientation angle and the distance travelled. Then there's vibration feedback at the two antennas. 

The second mode is track read. After travelling a recorded distance to reach the checkpoint, the tentacles on both sides will vibrate simultaneously. Touch the touch sensor to read the next set of azimuths and forward distances. One-sided tentacles vibrate to indicate the correct angle of steering.

## Content of Making Process
This file editing was done on Jialichuang's EasyEDA platform.
[JLC page of my project](https://www.https://https://u.easyeda.com/join?type=project&key=9ce47903dabc406eaa57221c2913183e&inviter=19f5151768ee47feb61f9181e9382d84/)

### Circuit Board Drawing
Schematic page
![picture description](./images/schematicpage.png)

PCB page
![picture description](./images/pcbpage.png)

### PCB soldering
![picture description](./images/pcb1.jpg)
![picture description](./images/pcb2.jpg)

### Source Code
```ruby
#include <Wire.h>
#include <MPU6050.h>
#include <SPI.h>
#include <SD.h>

MPU6050 mpu;
int steps = 0;
int flag = 0;
float threshold = 1.06;
float initialAngle = 0.0;
float currentAngle = 0.0;
float angularVelocity = 0.0;
unsigned long previousTime = 0;
File myFile;
int currentLine = 0;
int ModeSwicth = 3;
int touchPin = 4;
int LMotorPin = 5;
int RMotorPin = 6;
int Rsteps = 0;
int Rdirect = 0;

struct Data {
  int Rsteps;
  int Rdirect;
};
void setup() {
  Wire.begin();
  mpu.initialize();
  mpu.setDMPEnabled(true);
  initialAngle = calculateInitialAngle();
  previousTime = millis();
  Serial.begin(115200);

  pinMode(ModeSwicth, INPUT);
  pinMode(touchPin, INPUT);
  pinMode(LMotorPin, OUTPUT);
  pinMode(RMotorPin, OUTPUT);

  Serial.print("Initializing SD card...");
  if (!SD.begin(10)) {
    Serial.println("SD card initialization failed!");
    return;
  }
  Serial.println("SD card initialization done.");

  SD.remove("DATA.txt");
  myFile = SD.open("DATA.txt", FILE_WRITE);

  if (myFile) {
    Serial.println("File opened successfully.");
    myFile.close();
  }
}

void loop() {
  // Read accelerometer and gyroscope data
  int16_t ax, ay, az, gx, gy, gz;
  mpu.getMotion6(&ax, &ay, &az, &gx, &gy, &gz);

  // Calculate steps
  float aX = abs(ax) / 16384.0; // Convert to g
  if (aX >= threshold && flag == 0) {
    steps++;
    flag = 1;
  } else if (aX > threshold && flag == 1) {
    // Don’t Count
  }
  if (aX < threshold && flag == 1) {
    flag = 0;
  }

  // Calculate current angle
  unsigned long currentTime = millis();
  float elapsedTime = (currentTime - previousTime) / 1000.0;
  previousTime = currentTime;
  float angularVelocity = gx / 131.0;
  currentAngle = currentAngle + angularVelocity * elapsedTime;
  if (currentAngle > 180){
    currentAngle = currentAngle - 360;
  }
  if (currentAngle < -180){
    currentAngle = currentAngle + 360;
  }
  if (digitalRead(ModeSwicth) == HIGH) {
    currentAngle = currentAngle - 0.26;
    if (digitalRead(touchPin) == HIGH){
    digitalWrite(LMotorPin, HIGH);
    digitalWrite(RMotorPin, HIGH);
    writeDataToSDCard();
    Serial.print(steps);
    Serial.print(",");
    Serial.println(currentAngle - initialAngle);
    steps = 0;
    // delay(200);
    }
    else{
    digitalWrite(LMotorPin, LOW);
    digitalWrite(RMotorPin, LOW);
    }
    delay(200);
  }

  if (digitalRead(ModeSwicth) == LOW) {
    currentAngle = currentAngle - 0.5;
    if(digitalRead(touchPin) == HIGH){
    Data data = readDataFromSDCard();
    Rsteps = data.Rsteps;
    Rdirect = data.Rdirect;
    }

    // 计算角度差
    float angleDiff = currentAngle - Rdirect;
    if (abs(angleDiff) > 10 && angleDiff > 0){
    // 将角度差映射到震动电机的震动强度范围
    int vibrationStrength = map(angleDiff, 180, 0, 0, 255);
    // 限制震动强度在合理范围内
    vibrationStrength = constrain(vibrationStrength, 0, 255);
    // 控制震动电机
    analogWrite(LMotorPin, vibrationStrength);
    Serial.print(angleDiff);
    Serial.println("\t左转");
    delay(200);
    }

    if (abs(angleDiff) > 10 && angleDiff < 0){
    // 将角度差映射到震动电机的震动强度范围
    int vibrationStrength = map(angleDiff, -180, 0, 0, 255);
    // 限制震动强度在合理范围内
    vibrationStrength = constrain(vibrationStrength, 0, 255);
    // 控制震动电机
    analogWrite(RMotorPin, vibrationStrength);
    Serial.print(angleDiff);
    Serial.println("右转");
    delay(200);
    }

    if(steps == Rsteps){
    analogWrite(LMotorPin, 255);
    analogWrite(RMotorPin, 255);
    delay(200);
    steps = 0; 
    }
    else{
    analogWrite(LMotorPin, 0);
    analogWrite(RMotorPin, 0);
    }
  delay(200);
  }
Serial.print(digitalRead(ModeSwicth));
Serial.print("\t");
Serial.print(aX);
Serial.print("\t");
Serial.print(steps);
Serial.print("\t");
Serial.print(Rsteps);
Serial.print("\t");
Serial.println(currentAngle);
  // delay(100);
}

float calculateInitialAngle() {
  int16_t ax, ay, az;
  mpu.getAcceleration(&ax, &ay, &az);
  float initialAccelAngle = atan2(ay, az) * 180.0 / PI;
  return initialAccelAngle;
}
void writeDataToSDCard() {
  myFile = SD.open("data.txt", FILE_WRITE);
  if (myFile) {
    myFile.print(steps);
    myFile.print(",");
    myFile.println(currentAngle - initialAngle);
    myFile.close();
    Serial.println("Data written to SD card");
  } else {
    Serial.println("Failed to open file.");
  }
}
Data readDataFromSDCard(){
    Data result;

    // 打开文件
    myFile = SD.open("DATA.txt");
    if (myFile) {
      Serial.println("成功打开文件:");
      
      // 跳过已读取的行数
      for (int i = 0; i < currentLine; i++) {
        myFile.readStringUntil('\n');
      }

      // 读取下一行数据
      String line = myFile.readStringUntil('\n');
      Serial.println("读取的行数据: " + line);
      
      // 解析行数据，假设数据是用逗号分隔的
      result.Rsteps = line.substring(0, line.indexOf(',')).toInt();
      result.Rdirect = line.substring(line.indexOf(',') + 1).toInt();
      
      // 现在你可以使用result.Rsteps和result.Rdirect了
      Serial.print("Steps: ");
      Serial.println(result.Rsteps);
      Serial.print("Direct: ");
      Serial.println(result.Rdirect);
      
      myFile.close();
      // 增加当前行数，以便下一次读取下一行
      currentLine++;
    } else {
      Serial.println("无法打开文件.");
    }

    return result;
}
```
## Product Picture
Top-View
![picture description](./images/topview.jpg)

Front-View
![picture description](./images/frontview.jpg)

![picture description](./images/outdoor1.jpg)
![picture description](./images/outdoor2.jpg)

![picture description](./images/using.jpg)
