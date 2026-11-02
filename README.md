# ARDUINO
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
SoftwareSerial bt(11, 12);

const int trigPin  = 9;
const int echoPin = 8;
const int doorBtn = 2;
const int redPin  = 3;
const int greenPin= 5;
const int bluePin = 6;
const int buzzerPin= 4;
const int flamePin=A0;

const int TRIGGER_CM  = 10;      
const int FLAME_THRESH= 450;     
const unsigned long DEBOUNCE_MS = 40;

const unsigned long OPEN_AUTO_REARM_MS = 30000UL; // 30s auto rearm
unsigned long openedAtMs = 0;

enum SysState { DISARMED, ARMED, INTRUSION, OPENED };
SysState state = ARMED;

bool  sirenMuted = false;
bool  btnPrev = true;
unsigned long lastDebounce = 0;
long  lastDistCm = 0;

void setRGB(bool r, bool g, bool b) {
  digitalWrite(redPin,   r ? HIGH : LOW);
  digitalWrite(greenPin, g ? HIGH : LOW);
  digitalWrite(bluePin,  b ? HIGH : LOW);
}

void showLCD(const char* l1, const char* l2 = "") {
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print(l1);
  lcd.setCursor(0, 1); lcd.print(l2);
}

long getDistanceOnce() {
  digitalWrite(trigPin, LOW);  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH); delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long us =pulseIn(echoPin, HIGH, 30000);
  if (us == 0) return 0;
  return (us * 34L) / 2000L;
}

long getDistanceFiltered() {
  long a = getDistanceOnce(); delay(15);
  long b = getDistanceOnce(); delay(15);
  long c = getDistanceOnce();
  long avg = (a + b + c) / 3;
  if (avg > 0) lastDistCm= avg;
  return avg;
}

void alarmOn(const char* reasonBLE, const char* reasonLCD) {
  state = INTRUSION;
  sirenMuted = false;
  setRGB(true, false, false);// red
  digitalWrite(buzzerPin, HIGH); // buzzer on
  lcd.clear();
  lcd.setCursor(0,0); lcd.print("ALARM ACTIVE");
  lcd.setCursor(0,1);
  lcd.print(reasonLCD);
  lcd.print(" Dist:");
  lcd.print(lastDistCm);
  lcd.print("cm");
}

void alarmOffToOpened() {
  state =OPENED;
  openedAtMs = millis();// auto timer start
  digitalWrite(buzzerPin, LOW);
  setRGB(false, true, false); // green
  showLCD("INFO:OPENED", "Door unlocked");
}

void rearmToArmed() {
  state= ARMED;
  sirenMuted =false;
  setRGB(false, false, true);// blue
  digitalWrite(buzzerPin, LOW);
  showLCD("OK:ARMED", "ARMED (BLUE)");
}

void setup() {
  bt.begin(9600);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(doorBtn, INPUT_PULLUP);
  pinMode(redPin, OUTPUT);
  pinMode(greenPin, OUTPUT);
  pinMode(bluePin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  lcd.init();
  lcd.backlight();
  setRGB(false, false, true); // blue
  digitalWrite(buzzerPin, LOW);
  showLCD("System ARMED", "ARMED (BLUE)");
}

void loop() {
  if (bt.available()) {
    String cmd = bt.readStringUntil('\n');
    cmd.trim(); cmd.toUpperCase();
    if (cmd== "ARM") {
      rearmToArmed();
    } else if (cmd =="DISARM") {
      state = DISARMED; sirenMuted = false;
      setRGB(false, false, false);  // off
      digitalWrite(buzzerPin, LOW);
      showLCD("OK:DISARMED", "");
    } else if (cmd== "OPEN") {
      alarmOffToOpened(); // green + timer
    } else if (cmd =="SILENCE") {
      sirenMuted= true;
      digitalWrite(buzzerPin, LOW);
      showLCD("SIREN MUTED", "Await OPEN");
    }
  }

  bool btnNow = (digitalRead(doorBtn) == LOW);
  if (btnNow != btnPrev) { lastDebounce = millis(); btnPrev = btnNow; }
  if ((millis() - lastDebounce) > DEBOUNCE_MS) {
    if (btnNow && state== INTRUSION) {
      sirenMuted = true;
      digitalWrite(buzzerPin, LOW);
      showLCD("SIREN MUTED", "Await OPEN");
    }
  }

  if (state ==ARMED) {
    long d = getDistanceFiltered();
    if (d > 0 && d <= TRIGGER_CM) {
      alarmOn("INTRUSION", "INTRU");
    }
    int flameVal= analogRead(flamePin);
    if (flameVal < FLAME_THRESH) {
      alarmOn("FIRE", "FIRE NEAR DOOR");
    }
  }

  if (state ==OPENED && OPEN_AUTO_REARM_MS > 0) {
    if (millis() - openedAtMs >= OPEN_AUTO_REARM_MS) {
      rearmToArmed();
    }
  }

  if (state ==INTRUSION) {
    setRGB(true, false, false);// red
    digitalWrite(buzzerPin, sirenMuted ? LOW : HIGH);
  } else if (state== ARMED) {
    setRGB(false, false, true); // blue
    digitalWrite(buzzerPin, LOW);
  } else if (state ==OPENED) {
    setRGB(false, true, false);// green
    digitalWrite(buzzerPin, LOW);
  } else if (state== DISARMED) {
    setRGB(false, false, false); // off
    digitalWrite(buzzerPin, LOW);
  }

  delay(60);
}
