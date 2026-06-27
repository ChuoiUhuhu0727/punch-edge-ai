# punch-edge-ai

Wrist-worn device that classifies Aikido punch intensity and tracks heart rate recovery — **the Decision Tree runs fully on-chip** (ESP32-S3), no phone or cloud required. WiFi is used only at the end to email the session report.

**Course:** ENG209 Signals, Systems and Control — Fulbright University Vietnam  
**Team:** Hoàng Nguyễn Ngọc Giang · Phan Ngọc Quốc Duy · Trần Thanh Tùng

---

## What it does

| Input | Processing | Output |
|-------|-----------|--------|
| MPU6050 — 3-axis acceleration at 25 Hz | Sliding window (60 samples, stride 10) → 3 features → Decision Tree | **Normal** or **Intense** punch session label |
| MAX30102 — infrared PPG at 100 Hz | Moving average → IIR high-pass detrend → peak detection | Real-time BPM |
| 60 s rest window after punching stops | Peak BPM − Rest BPM | Heart Rate Recovery (HRR) score + cardiovascular rating |

The full session report (session class, peak BPM, rest BPM, HRR score) is emailed automatically via Gmail.

---

## System Pipeline

```
[MPU6050 25Hz]  ──► Acc_Mag = √(ax²+ay²+az²)
                     Sliding window 60 samples, stride 10
                     Features: peak_max, acc_std, peak_relative
                     Decision Tree (classifier.h)
                              │
                              ▼
                     NORMAL (0) or INTENSE (1)
                              │
[MAX30102 100Hz] ──► 5-pt moving avg → IIR high-pass
                     Peak detection → BPM
                     Cross-talk guard: freeze BPM if Acc_Mag > 14000 LSB (7g)
                              │
                              ▼
                     FSM: REST → PUNCHING → RECOVERY (60s) → DONE
                              │
                              ▼
                     Serial log + Gmail report (WiFi)
```

**FSM states:**
- `STATE_REST` — waiting for punch (Acc_Mag > 8192 LSB to trigger)
- `STATE_PUNCHING` — classifying windows, tracking peak BPM
- `STATE_RECOVERY` — 60-second heart rate cooldown window
- `STATE_DONE` — report sent; press `r` in Serial Monitor to reset

---

## Hardware

| Component | Role | Config |
|-----------|------|--------|
| Seeed Studio XIAO ESP32-S3 | Main MCU | 240 MHz, 512 KB SRAM |
| MPU6050 | IMU — acceleration + gyro | ±16g range, I2C 0x68 |
| MAX30102 | PPG — optical heart rate | IR 940 nm, I2C 0x57 |

**Wiring:** I2C shared bus — SDA = GPIO 5 (D4), SCL = GPIO 6 (D5), 400 kHz. Both sensors on 3.3 V.

---

## Signal Processing Details

### Heart Rate (PPG)
1. **Smoothing** — 5-point moving average over raw IR values
2. **Detrend** — IIR high-pass removes DC drift: `dc = 0.99·dc + 0.01·filtered`; `ac = filtered − dc`
3. **Peak detection** — state machine: rise above 150 → track peak; drop below −50 → record beat, compute BPM = 60000 / Δt
4. **Cross-talk guard** — if Acc_Mag > 14000 LSB, BPM update is frozen to reject motion artifacts

### Punch Classification (IMU)
1. **Magnitude** — collapse 3-axis to scalar: `Acc_Mag = √(ax²+ay²+az²)`
2. **Activity gate** — skip window if `acc_std < 1500 LSB` (device is still)
3. **Features** — `peak_max`, `acc_std`, `peak_relative = peak_max / mean`
4. **Decision Tree** — single split at `acc_std = 2488.65`; returns 0 (Normal) or 1 (Intense)
5. **Session label** — if ≥ 30% of windows are Intense → **INTENSE SESSION**

### I2C Watchdog
Hard impacts can lock the I2C bus. The watchdog detects this, resets the bus in 20 ms, and re-initialises both sensors. Max 3 auto-recovery attempts before raising a hardware fault flag and halting safely.

---

## AI Model

| Metric | Value |
|--------|-------|
| Model | Decision Tree (depth 3) |
| Features | `peak_max`, `acc_std`, `peak_relative` |
| Split threshold | `acc_std = 2488.65` |
| Accuracy (LOGO-CV) | 89.9% |
| Inference latency | < 5 µs on ESP32-S3 |
| RAM overhead | ~0 KB (embedded in `classifier.h`) |

---

## Setup

### Requirements
- [PlatformIO](https://platformio.org/) (CLI or VS Code extension)
- Seeed Studio XIAO ESP32-S3
- MPU6050 + MAX30102 wired as above
- Gmail account with an [App Password](https://myaccount.google.com/apppasswords) (2FA must be on)

### Configure credentials

Open [`firmware/main.cpp`](firmware/main.cpp) and fill in your values:

```cpp
#define WIFI_SSID      "your_wifi_name"
#define WIFI_PASSWORD  "your_wifi_password"
#define AUTHOR_EMAIL   "your_gmail@gmail.com"
#define AUTHOR_PASSWORD "xxxx xxxx xxxx xxxx"   // Gmail App Password
#define RECIPIENT_EMAIL "report_destination@gmail.com"
```

> The device works in **offline mode** if WiFi fails — it just skips the email step.

### Build and flash

```bash
# Flash production firmware (punch classifier + email report)
pio run -e seeed_xiao_esp32s3 -t upload

# Open Serial Monitor (115200 baud)
pio device monitor
```

---

## Results

10 test cases across functional accuracy, boundary values, robustness, and embedded performance:

| Category | Result |
|----------|--------|
| Intense session detection (acc_std > 3000) | PASSED |
| Normal session — not oversensitive (acc_std 1600–2200) | PASSED |
| Activity gate at 1500 LSB threshold | PASSED |
| Decision Tree boundary (acc_std 2400–2550) | PASSED |
| Motion-only (not a punch) not classified Intense | PASSED |
| Rotation invariance (abnormal wrist position) | PASSED |
| Rapid punching (3–4 punches/sec) — no missed windows | PASSED |
| Cross-talk guard freezes BPM during heavy impact | PASSED |
| FSM auto-renews after 3 s quiet period | PASSED |

**Known limitation:** If the user punches for less than 10 seconds, the body does not have time to raise heart rate — HRR score stays low (8 BPM range) even for a hard session. This is biological heart rate lag, not a code bug.

---

## Repo Structure

```
.
├── firmware/
│   ├── main.cpp          # Production firmware — punch classifier + HRR + email
│   └── classifier.h      # Decision Tree embedded as C header (generated from notebook)
├── data/
│   ├── raw/              # Raw CSV files from each collection session
│   └── processed/
│       └── master_dataset_aikido.csv   # 39k labelled samples
├── notebooks/
│   ├── 8th_Draft_Signal_Final_Project.ipynb   # Model training + classifier.h export
│   └── ...
├── scripts/
│   └── collect_data.py   # Serial data collector
├── docs/visual_qc/       # QC plots for each collection session
└── platformio.ini
```

---

## Demo

Video: [Google Drive](https://drive.google.com/file/d/1EUahLwN11yc4-B6_D1cpn5PuJAG3qsEr/view?usp=sharing)

Sample email output:
```
BÁO CÁO KẾT QUẢ ĐO LƯỜNG TỰ ĐỘNG TỪ THIẾT BỊ ĐEO
====================================================
Cường độ toàn hiệp: 🔥 HIỆP ĐẤM CƯỜNG ĐỘ CAO (INTENSE SESSION)
- Peak BPM  : 84.09
- Rest BPM  : 77.10
- HRR Score : 7.00 BPM
=> ĐÁNH GIÁ: XAU (Hệ tim mạch phục hồi kém)
```
