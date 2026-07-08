/*
 * Doctor Blade Coater — Fully Automated System v4 (Parallel LCD)
 * Speed: 0-45 mm/s | Return: Fixed 45 mm/s
 * Microstep: 1600 | L=8, GR=1.33
 * system constant of delay for 800 microstep is 6666 yani delay = 6666 / v ile bulunur.
 * eğer microstep 1600 olursa system constant of delay becomes 6666/2 = 3333 olur ve delay = 3333/ v ile bulunur.
 */

#include <LiquidCrystal.h>

// ── LCD SETTINGS ──────────────────────────────────────────────────────────────
const int rs = 12, en = 11, d4 = 10, d5 = 6, d6 = A1, d7 = A2;
LiquidCrystal lcd(rs, en, d4, d5, d6, d7);

// ── PIN DEFINITIONS ───────────────────────────────────────────────────────────
const int stepPin    = 9;
const int dirPin     = 13;
const int enPin      = 7;
const int potPin     = A0;
const int limit1Pin  = 3;  // Home Switch
const int limit2Pin  = 2;  // End Switch
const int button1Pin = 4;  // Start Button (Forward)
const int button2Pin = 5;  // Reset Button (Backward)

// ── SYSTEM VARIABLES ──────────────────────────────────────────────────────────
int systemState = 0; 
unsigned long lastDisplayTime = 0;
const unsigned long DISPLAY_INTERVAL = 300; // ms
float lockedV = 0.0;
long lockedStepDelay = 0;
int lastPotValue = 0; // Dalgalanma filtresi için eklendi

// ── SETUP ─────────────────────────────────────────────────────────────────────
void setup() {
  lcd.begin(16, 2);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("BGMAD Coater ");
  lcd.setCursor(0, 1);
  lcd.print("Starting...     ");
  delay(1500);
  lcd.clear();

  pinMode(stepPin,    OUTPUT);
  pinMode(dirPin,     OUTPUT);
  pinMode(enPin,      OUTPUT);

  pinMode(limit1Pin,  INPUT_PULLUP);
  pinMode(limit2Pin,  INPUT_PULLUP);
  pinMode(button1Pin, INPUT_PULLUP);
  pinMode(button2Pin, INPUT_PULLUP);

  digitalWrite(enPin, HIGH); // Başlangıçta motoru boşa al (sesi kes)
}

// ── MAIN LOOP ─────────────────────────────────────────────────────────────────
void loop() {
  bool btn1_pressed = (digitalRead(button1Pin) == LOW);
  bool btn2_pressed = (digitalRead(button2Pin) == LOW);
  bool lim1_pressed = (digitalRead(limit1Pin)  == LOW);
  bool lim2_pressed = (digitalRead(limit2Pin)  == LOW);

  switch (systemState) {

    case 0: // Waiting for Start
      {
        int potValue = analogRead(potPin);
        
        // Hysteresis (Deadband) Filtresi - Dalgalanmayı engeller
        if (abs(potValue - lastPotValue) > 3) {
            lastPotValue = potValue;
        }
        float currentV = lastPotValue * (45.0 / 1023.0);

        // Update LCD periodically to prevent garbage characters and lag
        if (millis() - lastDisplayTime > DISPLAY_INTERVAL) {
          lcd.setCursor(0, 0);
          lcd.print("Waiting Start...");
          lcd.setCursor(0, 1);
          lcd.print("Speed: ");
          if (currentV < 10.0) lcd.print(" ");
          lcd.print(currentV, 1);
          lcd.print(" mm/s ");
          lastDisplayTime = millis();
        }

        if (btn1_pressed && currentV > 0.1) {
          digitalWrite(enPin, LOW); // Harekete başlarken motoru aktif et
          lockedV = currentV;
          lockedStepDelay = 3333.0 / lockedV;
          digitalWrite(dirPin, HIGH);

          // Update LCD ONCE for State 1 to avoid motor stutter
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print("Coating...      ");
          lcd.setCursor(0, 1);
          lcd.print("Speed: ");
          if (lockedV < 10.0) lcd.print(" ");
          lcd.print(lockedV, 1);
          lcd.print(" mm/s ");

          systemState = 1;
          delay(300); // Debounce
        }
      }
      break;

    case 1: // Coating (Forward) - LCD is NOT updated here to prevent stutter
      if (lim2_pressed) {
        digitalWrite(enPin, HIGH); // Bitişte motoru boşa al (sesi kes)
        
        // Update LCD ONCE for State 2
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Coating Done    ");
        lcd.setCursor(0, 1);
        lcd.print("Waiting Reset...");
        systemState = 2;
      } else {
        moveStep(lockedStepDelay);
      }
      break;

    case 2: // Waiting for Reset
      if (btn2_pressed) {
        digitalWrite(enPin, LOW); // Harekete başlarken motoru aktif et
        digitalWrite(dirPin, LOW);

        // Update LCD ONCE for State 3
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Returning...    ");
        lcd.setCursor(0, 1);
        lcd.print("Speed: 45.0 mm/s");

        systemState = 3;
        delay(300); // Debounce
      }
      break;

    case 3: // Returning (Backward) - LCD is NOT updated here to prevent stutter
      if (lim1_pressed) {
        digitalWrite(enPin, HIGH); // Başlangıca dönünce motoru boşa al (sesi kes)
        lcd.clear();
        systemState = 0;
      } else {
        moveStep(3333 / 45); // 45 mm/s geri dönüş hızı
      }
      break;
  }
}

// ── HELPER FUNCTIONS ──────────────────────────────────────────────────────────
void moveStep(long delayTime) {
  digitalWrite(stepPin, HIGH);
  customDelay(delayTime);
  digitalWrite(stepPin, LOW);
  customDelay(delayTime);
}

void customDelay(long uSecs) {
  if (uSecs > 16000) {
    delay(uSecs / 1000);
    delayMicroseconds(uSecs % 1000);
  } else {
    delayMicroseconds(uSecs);
  }
}