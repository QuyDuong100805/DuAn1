#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Khai báo các chân kết nối LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

#define DIRX_PIN 22
#define PULX_PIN 23
#define CONTAC_X 24
#define POTX_PIN A1

#define DIRY_PIN 25
#define PULY_PIN 26
#define CONTAC_Y 27
#define POTY_PIN A2

#define DIRZ_PIN 28
#define PULZ_PIN 29
#define CONTAC_Z 30
#define POTZ_PIN A3

#define BUTTON_SAVE_PIN 31    // Chân của nút nhấn để lưu vị trí
#define BUTTON_RECALL_PIN 32  // Chân của nút nhấn để di chuyển về vị trí đã lưu
#define BUTTON_CLEAR_PIN 33  // Chân của nút nhấn để xoá các vị trí đã lưu

#define VACUUM_PIN 34         // Chân của máy bơm chân không

const int maxPositions = 10; // Số vị trí tối đa có thể lưu
int previousStepsX = 0;
int previousStepsY = 0;
int previousStepsZ = 0;

int savedStepsX[maxPositions] = {0};
int savedStepsY[maxPositions] = {0};
int savedStepsZ[maxPositions] = {0};

int currentPositionIndex = 0; // Chỉ số vị trí hiện tại
bool homingDone = false;
bool moveToSavedPositionFlag = false; // Thêm cờ để kiểm tra trạng thái di chuyển về vị trí đã lưu

void setup() {
  Serial.begin(9600); // Khởi tạo giao tiếp Serial với tốc độ 9600 bps

  pinMode(DIRX_PIN, OUTPUT);
  pinMode(PULX_PIN, OUTPUT);
  pinMode(POTX_PIN, INPUT);
  pinMode(CONTAC_X, INPUT_PULLUP);
  
  pinMode(DIRY_PIN, OUTPUT);
  pinMode(PULY_PIN, OUTPUT);
  pinMode(POTY_PIN, INPUT);
  pinMode(CONTAC_Y, INPUT_PULLUP);
  
  pinMode(DIRZ_PIN, OUTPUT);
  pinMode(PULZ_PIN, OUTPUT);
  pinMode(POTZ_PIN, INPUT);
  pinMode(CONTAC_Z, INPUT_PULLUP);
  
  pinMode(BUTTON_SAVE_PIN, INPUT_PULLUP); // Cấu hình chân nút nhấn để lưu vị trí
  pinMode(BUTTON_RECALL_PIN, INPUT_PULLUP); // Cấu hình chân nút nhấn để di chuyển về vị trí đã lưu
  pinMode(BUTTON_CLEAR_PIN, INPUT_PULLUP); // Cấu hình chân nút nhấn để xoá các vị trí đã lưu

  pinMode(VACUUM_PIN, OUTPUT); // Cấu hình chân máy bơm chân không

  // Khởi tạo LCD và hiển thị thông báo
  lcd.begin(16, 2);
  lcd.backlight(); // Bật đèn nền của màn hình LCD
  lcd.print("Homing...");

  // Di chuyển về vị trí home
  moveToHome();
  homingDone = true; // Đánh dấu quá trình homing đã hoàn tất

  // Xóa màn hình LCD và hiển thị thông báo
  lcd.clear();
  lcd.print("Ready");
}

void loop() {
  if (homingDone) {
    if (moveToSavedPositionFlag) {
      // Di chuyển về vị trí đã lưu nếu cờ được bật
      moveToSavedPosition();
      moveToSavedPositionFlag = false; // Tắt cờ sau khi hoàn thành di chuyển
    } else {
      // Đọc giá trị từ các biến trở và di chuyển động cơ tương ứng
      handleXAxis();
      handleYAxis();
      handleZAxis();
      delay(100); // Dừng lại giữa các lần đọc giá trị từ biến trở

      // Kiểm tra trạng thái của nút nhấn để lưu vị trí
      if (digitalRead(BUTTON_SAVE_PIN) == LOW) {
        rememberPosition();
        delay(100); // Chống nhiễu cho nút nhấn
      }

      // Kiểm tra trạng thái của nút nhấn để di chuyển về vị trí đã lưu
      if (digitalRead(BUTTON_RECALL_PIN) == LOW) {
        moveToSavedPositionFlag = true; // Bật cờ để di chuyển về vị trí đã lưu
        delay(100); // Chống nhiễu cho nút nhấn
      }

      // Kiểm tra trạng thái của nút nhấn để xoá các vị trí đã lưu
      if (digitalRead(BUTTON_CLEAR_PIN) == LOW) {
        clearPositions();
        delay(100); // Chống nhiễu cho nút nhấn
      }
    }
  }
}

// Hàm di chuyển về vị trí home
void moveToHome() {
  // Di chuyển trục X về home
  digitalWrite(DIRX_PIN, LOW); // Đặt HIGH để di chuyển về home
  while (digitalRead(CONTAC_X) == HIGH) {
    pulseMotor(PULX_PIN);
  }

  // Dừng lại sau khi về home
  delay(100);

  // Di chuyển trục Y và Z về home cùng lúc
  digitalWrite(DIRY_PIN, HIGH);  // Đặt LOW để di chuyển về home
  digitalWrite(DIRZ_PIN, HIGH); // Đặt HIGH để di chuyển về home
  while (digitalRead(CONTAC_Y) == HIGH || digitalRead(CONTAC_Z) == HIGH) {
    if (digitalRead(CONTAC_Y) == HIGH) {
      pulseMotor(PULY_PIN);
    }
    if (digitalRead(CONTAC_Z) == HIGH) {
      pulseMotor(PULZ_PIN);
    }
  }

  // Dừng lại sau khi về home
  delay(100);

  // Đặt hướng để di chuyển ra khỏi vị trí home
  digitalWrite(DIRX_PIN, HIGH);  // Đặt LOW để di chuyển ra khỏi vị trí home
  digitalWrite(DIRY_PIN, LOW); // Đặt HIGH để di chuyển ra khỏi vị trí home
  digitalWrite(DIRZ_PIN, LOW);  // Đặt LOW để di chuyển ra khỏi vị trí home
  
  // Sau khi về vị trí home, nhảy nhẹ ra một số bước
  moveSteps(DIRX_PIN, PULX_PIN, 300);
  moveSteps(DIRY_PIN, PULY_PIN, 200);
  moveSteps(DIRZ_PIN, PULZ_PIN, 1000);
}

// Hàm tạo xung cho động cơ
void pulseMotor(int pulPin) {
  digitalWrite(pulPin, HIGH);
  delayMicroseconds(100);  // Điều chỉnh tốc độ nếu cần
  digitalWrite(pulPin, LOW);
  delayMicroseconds(100);  // Điều chỉnh tốc độ nếu cần
}

// Hàm di chuyển một số bước xác định
void moveSteps(int dirPin, int pulPin, int steps) {
  for (int i = 0; i < steps; i++) {
    pulseMotor(pulPin);
  }
  delay(1100); // Độ trễ để động cơ ổn định
}

void handleXAxis() {
  int potValueX = analogRead(POTX_PIN); // Đọc giá trị từ biến trở X (0-1023)
  int stepsX = map(potValueX, 0, 1023, 0, 6400); // Chuyển đổi giá trị từ biến trở thành số bước quay

  // Nếu giá trị từ biến trở tăng, di chuyển động cơ X
  if (stepsX > previousStepsX) {
    int stepXDifference = stepsX - previousStepsX;
    digitalWrite(DIRX_PIN, HIGH); // Quay theo hướng ngược lại với homing
    for (int i = 0; i < stepXDifference; i++) {
      pulseMotor(PULX_PIN);
    }
    previousStepsX = stepsX;
  }

  // Hiển thị số bước động cơ X trên màn hình LCD
  lcd.setCursor(0, 0); // Đặt con trỏ tại dòng 0, cột 0
  lcd.print("X: ");
  lcd.print(stepsX);
  lcd.print("    "); // Xoá các ký tự thừa

  // In số bước của động cơ X ra Serial Monitor
  Serial.print("Bước X: ");
  Serial.println(stepsX);

  // Hiển thị thông tin từ Serial Monitor lên LCD
  updateLCDFromSerial();
}

void handleYAxis() {
  int potValueY = analogRead(POTY_PIN); // Đọc giá trị từ biến trở Y (0-1023)
  int stepsY = map(potValueY, 0, 1023, 0, 6400); // Chuyển đổi giá trị từ biến trở thành số bước quay

  // Nếu giá trị từ biến trở tăng, di chuyển động cơ Y
  if (stepsY > previousStepsY) {
    int stepYDifference = stepsY - previousStepsY;
    digitalWrite(DIRY_PIN, LOW); // Quay theo hướng ngược lại với homing
    for (int i = 0; i < stepYDifference; i++) {
      pulseMotor(PULY_PIN);
    }
    previousStepsY = stepsY;
  }

  // Hiển thị số bước động cơ Y trên màn hình LCD
  lcd.setCursor(0, 1); // Đặt con trỏ tại dòng 1, cột 0
  lcd.print("Y: ");
  lcd.print(stepsY);
  lcd.print("    "); // Xoá các ký tự thừa

  // In số bước của động cơ Y ra Serial Monitor
  Serial.print("Bước Y: ");
  Serial.println(stepsY);

  // Hiển thị thông tin từ Serial Monitor lên LCD
  updateLCDFromSerial();
}

void handleZAxis() {
  int potValueZ = analogRead(POTZ_PIN); // Đọc giá trị từ biến trở Z (0-1023)
  int stepsZ = map(potValueZ, 0, 1023, 0, 6400); // Chuyển đổi giá trị từ biến trở thành số bước quay

  // Nếu giá trị từ biến trở tăng, di chuyển động cơ Z
  if (stepsZ > previousStepsZ) {
    int stepZDifference = stepsZ - previousStepsZ;
    digitalWrite(DIRZ_PIN, LOW); // Quay theo hướng ngược lại với homing
    for (int i = 0; i < stepZDifference; i++) {
      pulseMotor(PULZ_PIN);
    }
    previousStepsZ = stepsZ;
  }

  // Hiển thị số bước động cơ Z trên màn hình LCD
  lcd.setCursor(8, 0); // Đặt con trỏ tại dòng 0, cột 8
  lcd.print("Z: ");
  lcd.print(stepsZ);
  lcd.print("    "); // Xoá các ký tự thừa

  // In số bước của động cơ Z ra Serial Monitor
  Serial.print("Bước Z: ");
  Serial.println(stepsZ);

  // Hiển thị thông tin từ Serial Monitor lên LCD
  updateLCDFromSerial();
}

void rememberPosition() {
  if (currentPositionIndex < maxPositions) {
    savedStepsX[currentPositionIndex] = previousStepsX;
    savedStepsY[currentPositionIndex] = previousStepsY;
    savedStepsZ[currentPositionIndex] = previousStepsZ;

    Serial.print("Đã lưu vị trí: ");
    Serial.print(currentPositionIndex + 1);
    Serial.print(" | X: ");
    Serial.print(savedStepsX[currentPositionIndex]);
    Serial.print(", Y: ");
    Serial.print(savedStepsY[currentPositionIndex]);
    Serial.print(", Z: ");
    Serial.println(savedStepsZ[currentPositionIndex]);

    currentPositionIndex++;

    // Hiển thị thông tin lên LCD
    lcd.clear();
    lcd.print("Saved Pos ");
    lcd.print(currentPositionIndex);
    lcd.setCursor(0, 1);
    lcd.print("X: ");
    lcd.print(savedStepsX[currentPositionIndex - 1]);
    lcd.print(" Y: ");
    lcd.print(savedStepsY[currentPositionIndex - 1]);
    lcd.print(" Z: ");
    lcd.print(savedStepsZ[currentPositionIndex - 1]);
  } else {
    Serial.println("Không thể lưu thêm vị trí, đã đạt giới hạn.");

    // Hiển thị thông tin lên LCD
    lcd.clear();
    lcd.print("Limit Reached");
  }
}

void moveToSavedPosition() {
  if (currentPositionIndex > 0) {
    for (int index = 0; index < currentPositionIndex; index++) {
      // Di chuyển về home giữa các vị trí đã lưu
      moveToHome();

      // Di chuyển trục X trước
      moveMotorToPosition(DIRX_PIN, PULX_PIN, savedStepsX[index], HIGH);

      // Di chuyển trục Z tiếp theo
      moveMotorToPosition(DIRZ_PIN, PULZ_PIN, savedStepsZ[index], LOW);

      // Di chuyển trục Y cuối cùng
      moveMotorToPosition(DIRY_PIN, PULY_PIN, savedStepsY[index], LOW);

      // Kích hoạt máy bơm chân không nếu vị trí là 1, 3, 5, 7, 9
      if ((index + 1) % 2 != 0) {
        digitalWrite(VACUUM_PIN, HIGH); // Kích hoạt máy bơm chân không
      } else {
        digitalWrite(VACUUM_PIN, LOW); // Tắt máy bơm chân không
      }

      Serial.print("Di chuyển đến vị trí đã lưu: ");
      Serial.print(index + 1);
      Serial.print(" | X: ");
      Serial.print(savedStepsX[index]);
      Serial.print(", Y: ");
      Serial.print(savedStepsY[index]);
      Serial.print(", Z: ");
      Serial.println(savedStepsZ[index]);

      // Hiển thị thông tin lên LCD
      lcd.clear();
      lcd.print("Moving to Pos ");
      lcd.print(index + 1);
      lcd.setCursor(0, 1);
      lcd.print("X: ");
      lcd.print(savedStepsX[index]);
      lcd.print(" Y: ");
      lcd.print(savedStepsY[index]);
      lcd.print(" Z: ");
      lcd.print(savedStepsZ[index]);
      
      // Tắt máy bơm chân không sau khi di chuyển đến các vị trí 2, 4, 6, 8, 10
      if ((index + 1) % 2 == 0) {
        digitalWrite(VACUUM_PIN, LOW); // Tắt máy bơm chân không
      }
    }
  } else {
    Serial.println("Không có vị trí nào để di chuyển đến.");

    // Hiển thị thông tin lên LCD
    lcd.clear();
    lcd.print("No Position");
  }
}

void moveMotorToPosition(int dirPin, int pulPin, int steps, int direction) {
  digitalWrite(dirPin, direction); // Đặt hướng di chuyển
  for (int i = 0; i < steps; i++) {
    pulseMotor(pulPin);
  }
  delay(1100); // Độ trễ để động cơ ổn định
}

void clearPositions() {
  currentPositionIndex = 0; // Đặt lại chỉ số vị trí hiện tại
  for (int i = 0; i < maxPositions; i++) {
    savedStepsX[i] = 0;
    savedStepsY[i] = 0;
    savedStepsZ[i] = 0;
  }
  Serial.println("Đã xoá tất cả các vị trí đã lưu.");

  // Hiển thị thông tin lên LCD
  lcd.clear();
  lcd.print("All Positions");
  lcd.setCursor(0, 1);
  lcd.print("Cleared");
}

// Hàm cập nhật LCD từ Serial Monitor
void updateLCDFromSerial() {
  while (Serial.available()) {
    char ch = Serial.read();
    lcd.write(ch);
  }
}
