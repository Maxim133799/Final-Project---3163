#include <SPI.h>
#include <MFRC522.h>
#include <time.h>
#include <math.h>
#include <Preferences.h>

// ========================
// Project sizing
// ========================
#define NUM_OF_SWITCHES   4
#define NUM_OF_SOLENOIDS  4
#define NUM_OF_RFIDS      4
#define NUM_OF_LAPTOPS    4

// ========================
// Student card reader (RDM6300 over UART)
// ========================
#define RDM6300_RX_PIN 17
HardwareSerial RFIDSerial(1);

// ========================
// Slot RC522 readers (ESP32-C6 SPI pinning)
// ========================
static const int PIN_SCK  = 6;   // SCK  (FSPI CLK)  ← don't reuse elsewhere
static const int PIN_MOSI = 7;   // MOSI (FSPI D)    ← don't reuse elsewhere
static const int PIN_MISO = 2;   // MISO (FSPI Q)

#define RC522_RST 14
#define RC522_SS0 8
#define RC522_SS1 9
#define RC522_SS2 10
#define RC522_SS3 11

MFRC522 rfid0(RC522_SS0, RC522_RST);
MFRC522 rfid1(RC522_SS1, RC522_RST);
MFRC522 rfid2(RC522_SS2, RC522_RST);
MFRC522 rfid3(RC522_SS3, RC522_RST);
MFRC522* rfids[NUM_OF_RFIDS] = { &rfid0, &rfid1, &rfid2, &rfid3 };

// ========================
// Solenoid DEMUX pins
// ========================
#define SolSelA 21
#define SolSelB 20
#define SolSelC 19

// ========================
// Door switch mux pins  (ADJUSTED to avoid SPI)
// ========================
#define SwitchPin  5      // mux common output into ESP32
#define SwitchSelA 22     // mux select A
#define SwitchSelB 23     // mux select B

// Switch pressed = DOOR CLOSED (adjust if wired opposite)
#define SWITCH_PRESSED_STATE  HIGH

// ========================
// Laptop sticker UIDs (expected per slot)
// ========================
String laptopUIDs[NUM_OF_LAPTOPS] = {
  "6AEE64F6",  // slot 0  (replace with your actual UIDs)
  "DABF64F6",  // slot 1
  "FFFFFFFF",  // slot 2
  "FFFFFFFF"   // slot 3
};

// ========================
// Persistence & data structures (unchanged)
// ========================
Preferences prefs;

typedef struct {
  bool    free;
  int     laptopNum;
  int     ID;        // student ID as int (parsed from hex string)
  time_t  loanTime;
  int     logIndex;
} laptop;

typedef struct {
  int     laptopNum;
  int     ID;
  time_t  loanTime;
  time_t  returnTime;
} dataLog;

laptop*  laptops  = nullptr;
dataLog* loanLog  = nullptr;
int      logIndex = 0;

bool switchStateCurr[NUM_OF_SWITCHES] = { true, true, true, true };

// ========================
// Automatic checkup timing
// ========================
unsigned long lastCheckupTime = 0;
const unsigned long CHECKUP_INTERVAL = 600000; // 10 minutes in milliseconds

// ========================
// Forward decls
// ========================
void   backupStateToFlash();
void   loadStateFromFlash();
void   updateLog(int laptopNum);
void   loanLaptop(int laptopNum, int ID);
void   returnLaptop(int laptopNum);

void   openSolenoid(unsigned int solNum);
int    waitForStudentCard();

void   readSwitch(int idx);
void   readAllSwitches();
void   printSwitchStates();

void   checkSlots();
bool   readUID_NoRemove(MFRC522 &r, String &uidOut);

void   printSection(const char* title);

// ========================
// Setup
// ========================
void setup() {
  Serial.begin(115200);
  delay(200);

  // Student card reader (RDM6300)
  RFIDSerial.begin(9600, SERIAL_8N1, RDM6300_RX_PIN, -1);
  Serial.println("Menu: '1' Loan | '2' Return");

  // Slot RC522 readers (SPI)
  SPI.begin(PIN_SCK, PIN_MISO, PIN_MOSI);
  pinMode(RC522_SS0, OUTPUT); digitalWrite(RC522_SS0, HIGH);
  pinMode(RC522_SS1, OUTPUT); digitalWrite(RC522_SS1, HIGH);
  pinMode(RC522_SS2, OUTPUT); digitalWrite(RC522_SS2, HIGH);
  pinMode(RC522_SS3, OUTPUT); digitalWrite(RC522_SS3, HIGH);

  for (int i = 0; i < NUM_OF_RFIDS; i++) {
    rfids[i]->PCD_Init();
    delay(50);
  }

  // Switch mux
  pinMode(SwitchPin, INPUT_PULLUP);        // use INPUT or INPUT_PULLUP per your wiring
  pinMode(SwitchSelA, OUTPUT);
  pinMode(SwitchSelB, OUTPUT);

  // Solenoids
  pinMode(SolSelA, OUTPUT);
  pinMode(SolSelB, OUTPUT);
  pinMode(SolSelC, OUTPUT);
  digitalWrite(SolSelA, HIGH);
  digitalWrite(SolSelB, HIGH);
  digitalWrite(SolSelC, HIGH);

  loadStateFromFlash();

  Serial.println("Setup completed.");
}

// ========================
// Loop (keyboard menu with automatic checkup)
// ========================
void loop() {
  // Check if it's time for automatic checkup
  unsigned long currentTime = millis();
  if (currentTime - lastCheckupTime >= CHECKUP_INTERVAL) {
    printSection("Automatic checkup started");
    checkSlots();           // RFID + switches
    printSection("Automatic checkup ended");
    lastCheckupTime = currentTime;
  }

  if (Serial.available()) {
    char cmd = Serial.read();

    if (cmd == '1') {
      printSection("Loan process started");
      Serial.println("Scan student card (5s timeout)...");
      int studentID = waitForStudentCard();
      if (studentID == -1) {
        Serial.println("Timeout waiting for student card.");
        printSection("Loan process ended");
        return;
      }

      int freeSlot = -1;
      for (int i = 0; i < NUM_OF_LAPTOPS; i++) {
        if (laptops[i].free) { freeSlot = i; break; }
      }
      if (freeSlot == -1) {
        Serial.println("There are no available laptops at the moment.");
        printSection("Loan process ended");
        return;
      }

      loanLaptop(freeSlot, studentID);
      Serial.printf("Laptop %d assigned to student %d\n", freeSlot, studentID);
      openSolenoid(freeSlot);
      printSection("Loan process ended");
    }

    else if (cmd == '2') {
      printSection("Return process started");
      bool anyTaken = false;
      for (int i = 0; i < NUM_OF_LAPTOPS; i++) {
        if (!laptops[i].free) { anyTaken = true; break; }
      }
      if (!anyTaken) {
        Serial.println("No laptops are currently loaned out.");
        printSection("Return process ended");
        return;
      }

      Serial.println("Scan student card (5s timeout)...");
      int studentID = waitForStudentCard();
      if (studentID == -1) {
        Serial.println("Timeout waiting for student card.");
        printSection("Return process ended");
        return;
      }

      int slot = -1;
      for (int i = 0; i < NUM_OF_LAPTOPS; i++) {
        if (!laptops[i].free && laptops[i].ID == studentID) { slot = i; break; }
      }
      if (slot == -1) {
        Serial.println("Student didn't loan a laptop.");
        printSection("Return process ended");
        return;
      }

      returnLaptop(slot);
      Serial.printf("Laptop %d returned. Please place in the slot.\n", slot);
      openSolenoid(slot);
      printSection("Return process ended");
    }
  }
}

// ========================
// Pretty separators
// ========================
void printSection(const char* title) {
  Serial.println();
  Serial.println("================================");
  Serial.println(title);
  Serial.println("================================");
  Serial.println();
}

// ========================
// Student card reading (RDM6300)
// ========================
int waitForStudentCard() {
  unsigned long start = millis();
  while (millis() - start < 5000) {
    if (RFIDSerial.available() >= 14) {
      if (RFIDSerial.read() == 0x02) {
        char tagData[13];
        RFIDSerial.readBytes(tagData, 12);
        tagData[12] = '\0';
        RFIDSerial.read(); // checksum
        RFIDSerial.read(); // ETX

        // Clear repeats and require removal quiet time
        unsigned long removeStart = millis();
        while (millis() - removeStart < 2000) {
          if (RFIDSerial.available() >= 14) {
            while (RFIDSerial.available()) RFIDSerial.read();
            removeStart = millis();
          }
        }
        return (int)strtoul(tagData, NULL, 16);
      }
    }
  }
  return -1; // timeout
}

// ========================
// Slot checkup (RFID + door switches)
// ========================
bool readUID_NoRemove(MFRC522 &r, String &uidOut) {
  // Normal path
  if (r.PICC_IsNewCardPresent() && r.PICC_ReadCardSerial()) {
    String uid="";
    for (byte j=0; j<r.uid.size; j++) { if (r.uid.uidByte[j] < 0x10) uid += "0"; uid += String(r.uid.uidByte[j], HEX); }
    uid.toUpperCase();
    uidOut = uid;
    r.PICC_HaltA();
    r.PCD_StopCrypto1();
    return true;
  }
  // Wakeup path (tag still in field but halted)
  byte atqa[2]; byte atqaSize = sizeof(atqa);
  MFRC522::StatusCode st = r.PICC_WakeupA(atqa, &atqaSize);
  if (st == MFRC522::STATUS_OK && r.PICC_ReadCardSerial()) {
    String uid="";
    for (byte j=0; j<r.uid.size; j++) { if (r.uid.uidByte[j] < 0x10) uid += "0"; uid += String(r.uid.uidByte[j], HEX); }
    uid.toUpperCase();
    uidOut = uid;
    r.PICC_HaltA();
    r.PCD_StopCrypto1();
    return true;
  }
  return false;
}

void checkSlots() {
  Serial.println("Checking available slots and verifying laptops...");

  // ---- RFID per free slot ----
  for (int i = 0; i < NUM_OF_LAPTOPS; i++) {
    if (!laptops[i].free) continue;  // only verify slots that should have a laptop

    if (laptopUIDs[i].length() > 0) {
      Serial.printf("Slot %d expected UID: %s\n", i, laptopUIDs[i].c_str());
    } else {
      Serial.printf("Slot %d expected UID: (not set)\n", i);
    }

    String uid;
    if (readUID_NoRemove(*rfids[i], uid)) {
      Serial.printf("Reader %d: UID = %s\n", i, uid.c_str());
      if (laptopUIDs[i].length() > 0) {
        if (uid == laptopUIDs[i]) {
          Serial.printf("Slot %d: Correct laptop present.\n", i);
        } else {
          Serial.printf("Slot %d: WRONG laptop! expected %s\n", i, laptopUIDs[i].c_str());
        }
      }
    } else {
      Serial.printf("No tag detected on reader %d\n", i);
    }
  }

  // ---- Door switches (all slots) ----
  Serial.println();
  Serial.println("Door switch states:");
  for (int i = 0; i < NUM_OF_SWITCHES; i++) {
    readSwitch(i);
    bool closed = (switchStateCurr[i] == SWITCH_PRESSED_STATE);
    Serial.printf("Slot %d door: %s\n", i, closed ? "CLOSED" : "OPEN");
  }
}

// ========================
// Switch helpers
// ========================
void readSwitch(int switchNum) {
  if (switchNum < 0 || switchNum >= NUM_OF_SWITCHES) {
    Serial.println("Invalid switch number.");
    return;
  }
  digitalWrite(SwitchSelA, (switchNum & 1));
  digitalWrite(SwitchSelB, ((switchNum >> 1) & 1));
  delay(60);  // small settle time for mux
  switchStateCurr[switchNum] = digitalRead(SwitchPin);
}

void readAllSwitches() {
  for (int i = 0; i < NUM_OF_SWITCHES; i++) readSwitch(i);
}

void printSwitchStates() {
  Serial.print("switch_state {");
  for (int i = 0; i < NUM_OF_SWITCHES; i++) {
    Serial.print(switchStateCurr[i]);
    Serial.print((i < NUM_OF_SWITCHES-1) ? ", " : "");
  }
  Serial.println("}");
}

// ========================
// Solenoids
// ========================
void openSolenoid(unsigned int solNum) {
  if (solNum > NUM_OF_SOLENOIDS - 1) return;
  digitalWrite(SolSelA,  (solNum & 1));
  digitalWrite(SolSelB, ((solNum >> 1) & 1));
  digitalWrite(SolSelC, ((solNum >> 2) & 1));  // 0..3 → LOW enable on your board
  delay(3000);
  digitalWrite(SolSelC, HIGH);
  delay(200);
}

// ========================
// Persistence & logging
// ========================
void updateLog(int laptopNum) {
  if (laptops[laptopNum].free) {
    if (laptops[laptopNum].logIndex >= 0) {
      time(&loanLog[laptops[laptopNum].logIndex].returnTime);
    }
  } else {
    loanLog[logIndex].laptopNum = laptopNum;
    loanLog[logIndex].ID        = laptops[laptopNum].ID;
    loanLog[logIndex].loanTime  = laptops[laptopNum].loanTime;
    logIndex++;
    dataLog* tmp = (dataLog*)realloc(loanLog, (logIndex + 1) * sizeof(dataLog));
    if (tmp != nullptr) {
      loanLog = tmp;
      loanLog[logIndex].laptopNum  = -1;
      loanLog[logIndex].ID         = -1;
      loanLog[logIndex].loanTime   = 0;
      loanLog[logIndex].returnTime = 0;
    } else {
      Serial.println("Failed to expand loanLog!");
    }
  }
}

void loanLaptop(int laptopNum, int ID) {
  laptops[laptopNum].free = false;
  laptops[laptopNum].ID   = ID;
  time(&laptops[laptopNum].loanTime);
  laptops[laptopNum].logIndex = logIndex;
  updateLog(laptopNum);
  backupStateToFlash();
}

void returnLaptop(int laptopNum) {
  laptops[laptopNum].free = true;
  updateLog(laptopNum);
  laptops[laptopNum].ID       = -1;
  laptops[laptopNum].loanTime = 0;
  laptops[laptopNum].logIndex = -1;
  backupStateToFlash();
}

void backupStateToFlash() {
  prefs.begin("storage", false);
  prefs.putInt("logIndex", logIndex);
  prefs.putBytes("loanLog", loanLog, (logIndex + 1) * sizeof(dataLog));
  prefs.putBytes("laptops", laptops, NUM_OF_LAPTOPS * sizeof(laptop));
  prefs.end();
}

void loadStateFromFlash() {
  prefs.begin("storage", true);

  logIndex = prefs.getInt("logIndex", 0);

  if (laptops) free(laptops);
  laptops = (laptop*)calloc(NUM_OF_LAPTOPS, sizeof(laptop));
  for (int i = 0; i < NUM_OF_LAPTOPS; i++) {
    laptops[i].free      = true;
    laptops[i].laptopNum = i;
    laptops[i].ID        = -1;
    laptops[i].loanTime  = 0;
    laptops[i].logIndex  = -1;
  }
  prefs.getBytes("laptops", laptops, NUM_OF_LAPTOPS * sizeof(laptop));

  if (loanLog) free(loanLog);
  loanLog = (dataLog*)calloc(logIndex + 1, sizeof(dataLog));
  for (int i = 0; i <= logIndex; i++) {
    loanLog[i].laptopNum  = -1;
    loanLog[i].ID         = -1;
    loanLog[i].loanTime   = 0;
    loanLog[i].returnTime = 0;
  }
  prefs.getBytes("loanLog", loanLog, (logIndex + 1) * sizeof(dataLog));

  prefs.end();
}
