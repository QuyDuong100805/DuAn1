#include <Wire.h>
#include <LiquidCrystal_I2C_STEM.h>
#include <Servo.h>

// Khai báo các chân
const int TRIG_PIN = 7;
const int ECHO_PIN = 6;
const int IR_PIN1 = 8;  // Cảm biến hồng ngoại 1 (dừng băng tải)
const int SERVO_PIN = 10;
const int BUTTON_PIN = 11;
const int BUZZER_PIN = 12;
const int CONVEYOR_MOTOR_PIN = 13;
const int STATUS_LED_PIN = 5;
const int STOP_BUTTON_PIN = 14;
const int STOP_LED_PIN = 15;
const int DISTANCE_THRESHOLD = 6;  // Giới hạn khoảng cách để phân loại sản phẩm

// Khai báo LCD, Servo
LiquidCrystal_I2C_STEM lcd(0x27, 16, 2);
Servo myServo;

int objectCount = 0;
int errorCount = 0;
bool irState1 = false;
bool prevIrState1 = false;
bool systemStarted = false;

void setup() {
  Serial.begin(9600);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(IR_PIN1, INPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(STOP_BUTTON_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(CONVEYOR_MOTOR_PIN, OUTPUT);
  pinMode(STATUS_LED_PIN, OUTPUT);
  pinMode(STOP_LED_PIN, OUTPUT);

  lcd.begin(16, 2);
  lcd.backlight();
  lcd.setCursor(0, 0);
  
  myServo.attach(SERVO_PIN);
}

void loop() {
  handleStopButton();
  handleStartButton();
  if (systemStarted) {
    handleUltrasonicSensor();
    handleIRSensors();
  }
}

void handleStopButton() {
  if (digitalRead(STOP_BUTTON_PIN) == LOW) {
    Serial.println("Hệ thống đã dừng do nhấn nút stop.");
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("He thong da dung");
    
    digitalWrite(STATUS_LED_PIN, LOW);
    digitalWrite(CONVEYOR_MOTOR_PIN, LOW);
    digitalWrite(STOP_LED_PIN, HIGH);
    
    while (digitalRead(STOP_BUTTON_PIN) == LOW) {
    }

    systemStarted = false;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Nhe nhan START de");
    lcd.setCursor(0, 1);
    lcd.print("bat lai he thong");
  }
}

void handleStartButton() {
  if (!systemStarted && digitalRead(BUTTON_PIN) == LOW) {
    delay(100);
    if (digitalRead(BUTTON_PIN) == LOW) {
      systemStarted = true;
      Serial.println("Hệ thống đã khởi động");
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("He thong chay");
      
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(CONVEYOR_MOTOR_PIN, HIGH);
      digitalWrite(STATUS_LED_PIN, HIGH);
      digitalWrite(STOP_LED_PIN, LOW);  // Tắt đèn stop khi khởi động lại hệ thống
      delay(5000);
      digitalWrite(BUZZER_PIN, LOW);
      
      delay(1000);
      lcd.clear();
    }
  }
}

void handleUltrasonicSensor() {
  long duration_us;
  float distance_cm;

  // Gửi tín hiệu kích hoạt
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2); 
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  // Đo thời gian phản hồi
  duration_us = pulseIn(ECHO_PIN, HIGH, 30000);  // timeout sau 30ms nếu không nhận được phản hồi
  distance_cm = duration_us * 0.034 / 2;

  Serial.print("Khoang cach: ");
  Serial.println(distance_cm);

  // Không xử lý nếu khoảng cách lớn hơn 9 cm
  if (distance_cm > 9 || distance_cm <= 0) {
    return;
  }

  if (distance_cm > DISTANCE_THRESHOLD) {
    // Khoảng cách lớn hơn 5 cm, phôi lỗi
    errorCount++;
    Serial.print("Phôi lỗi: ");
    Serial.println(errorCount);

    lcd.setCursor(0, 0);
    lcd.print("SP dat: ");
    lcd.print(objectCount);
    lcd.setCursor(0, 1);
    lcd.print("Phoi loi: ");
    lcd.print(errorCount);

    // Điều khiển servo gạt phôi lỗi
    myServo.write(80);  // Đặt servo ở góc 80 độ
    delay(2500);
  } else {
    // Khoảng cách nhỏ hơn hoặc bằng 5 cm, sản phẩm đạt
    objectCount++;
    Serial.print("Sản phẩm đạt: ");
    Serial.println(objectCount);

    lcd.setCursor(0, 0);
    lcd.print("SP dat: ");
    lcd.print(objectCount);
    lcd.setCursor(0, 1);
    lcd.print("Phoi loi: ");
    lcd.print(errorCount);

    // Điều khiển servo về góc 0 độ cho sản phẩm đạt
    myServo.write(0);  // Đặt servo ở góc 0 độ
    delay(2500);
  }
}

void handleIRSensors() {
  irState1 = digitalRead(IR_PIN1);

  // Đọc trạng thái của cảm biến hồng ngoại 1
  Serial.print("IR1: ");
  Serial.println(irState1);

  // Nếu cảm biến hồng ngoại 1 phát hiện vật thể, dừng băng tải
  if (irState1 == HIGH) {
    digitalWrite(CONVEYOR_MOTOR_PIN, LOW);
    Serial.println("Dung bang tai do phat hien vat tu IR1.");

    lcd.setCursor(0, 0);
    lcd.print("Dung bang tai: ");


    // Đợi cho đến khi vật thể biến mất
    while (digitalRead(IR_PIN1) == HIGH) {
    }

    // Tiếp tục băng tải khi vật thể đã biến mất
    digitalWrite(CONVEYOR_MOTOR_PIN, HIGH);
    Serial.println("Tiep tuc bang tai sau khi vat tu bien mat.");

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Tiep tuc bang tai");
  }
}
