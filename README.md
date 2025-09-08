# Laptop Lending Station Project

## Overview
This project implements a smart **Laptop Lending Station** using the ESP32-C6.  
The system manages laptop loans and returns, verifies correct laptop placement using RFID, and ensures secure door operation with solenoids and switches.

---

## Features
- **Student Card Reader (RDM6300, UART):** Identifies students borrowing/returning laptops.
- **Slot RFID Readers (MFRC522, SPI):** Verify correct laptops are placed in the correct slots.
- **Solenoid Locks:** Securely open compartments for loan and return.
- **Door Switches (via MUX):** Detect whether doors are closed properly.
- **Data Logging (Flash Storage):** Tracks loan/return history with timestamps.
- **Automatic Checkup Routine:** Every 10 minutes the system verifies laptop presence and door closure.
- **Scalability:** Use of MUX/DEMUX reduces GPIO consumption, making it easy to extend with more slots.

---

## Hardware Components
- ESP32-C6 DevKit
- RDM6300 RFID Reader (UART, 125 kHz)
- 4 × MFRC522 RFID Readers (SPI, 13.56 MHz)
- Solenoids + MOSFET driver board
- CD4051B MUX for switches
- Power supply (12V for solenoids, 3.3V/5V for logic)

---

## Pin Assignments
| Component                | ESP32 Pin |
|---------------------------|-----------|
| RDM6300 RX (student card) | 17        |
| MFRC522 SCK               | 6         |
| MFRC522 MOSI              | 7         |
| MFRC522 MISO              | 2         |
| MFRC522 RST               | 14        |
| MFRC522 SS0               | 8         |
| MFRC522 SS1               | 9         |
| MFRC522 SS2               | 10        |
| MFRC522 SS3               | 11        |
| Solenoid SelA             | 21        |
| Solenoid SelB             | 20        |
| Solenoid SelC             | 19        |
| Switch MUX Out            | 5         |
| Switch SelA               | 22        |
| Switch SelB               | 23        |

---

## Installation Instructions
1. Install the **Arduino IDE** (latest version).
2. In Arduino IDE, install the following libraries from the Library Manager:
   - `MFRC522` (Version 1.4.10 or later)
   - `Preferences` (included in ESP32 board package)
   - `SPI` (included in Arduino core)
   - `time` (included in ESP32 board package)
3. Install the ESP32 board support in Arduino IDE:
   - Go to *File → Preferences → Additional Board Manager URLs*
   - Add: `https://dl.espressif.com/dl/package_esp32_index.json`
   - Then install **ESP32 by Espressif Systems** via *Tools → Board → Board Manager*

---

## Usage
1. Power the ESP32-C6 with USB (and 12V for solenoids).
2. Open Serial Monitor at **115200 baud**.
3. Enable CDC under tools
4. Press:
   - `1` → Loan process (assigns laptop if available).
   - `2` → Return process (validates and frees laptop).
5. Every 10 minutes, an **automatic checkup** runs to verify:
   - All free slots have the correct laptop (via RFID).
   - All doors are closed (via switches).

---

## Limitations
- Solenoids consume **high current** if energized too long → can overheat if not managed.
- Loan/return times in logs currently depend on ESP32’s unsynced clock (not accurate without RTC or WiFi NTP).
- ID read from RFID reader cannot be verified as a university issued card.
- Currently controlled via **Serial Monitor** (keyboard input), no GUI yet.

---

## Future Improvements
- Add **WiFi NTP synchronization** for accurate loan/return timestamps.
- Add **HTML page or GUI** for management.
- Optimize **power management** for solenoids (timed relay cutoff).
- Expand to more slots using larger MUX/DEMUX configurations.

---

## Authors
- Project Team: Maxim Shapira & Ohad Solovatshik
- Supervisor: Hezi Nistani
- Location: Tel Aviv University 
