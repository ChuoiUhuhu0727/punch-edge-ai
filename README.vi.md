# punch-edge-ai

🇬🇧 [Read in English](README.md)

Thiết bị đeo cổ tay cho việc tập Aikido — phân loại từng đòn đấm là Thường hay Mạnh và theo dõi hồi phục nhịp tim. **Mô hình AI chạy trực tiếp trên chip ESP32-S3** — không cần điện thoại hay cloud. WiFi chỉ dùng ở bước cuối để gửi báo cáo qua email.

**Môn học:** ENG209 Tín hiệu, Hệ thống và Điều khiển — Đại học Fulbright Việt Nam  
**Nhóm:**  
[Hoàng Nguyễn Ngọc Giang](https://www.linkedin.com/in/giang-ho%C3%A0ng/) — Xử lý tín hiệu & AI  
[Phan Ngọc Quốc Duy](https://www.linkedin.com/in/duy-phan-ngọc-quốc-3a342a312) — Phần cứng  
[Trần Thanh Tùng](https://www.linkedin.com/in/t%C3%B9ng-tr%E1%BA%A7n/) — Phần cứng

---

## Thiết bị làm gì?

| Đầu vào | Xử lý | Đầu ra |
|---------|-------|--------|
| MPU6050 — gia tốc 3 trục @ 25 Hz | Sliding window → Decision Tree | Hiệp đấm **Thường** hay **Mạnh** |
| MAX30102 — PPG hồng ngoại @ 100 Hz | Moving avg → IIR detrend → phát hiện đỉnh | Nhịp tim BPM theo thời gian thực |
| 60 giây nghỉ sau khi ngừng đấm | BPM đỉnh − BPM lúc nghỉ | Chỉ số hồi phục tim (HRR) |

![Dữ liệu đầu vào thô](docs/images/raw%20input%20data.png)
*Tín hiệu thô từ cảm biến: Acc\_Mag từ MPU6050 (độ lớn gia tốc) và IR PPG từ MAX30102 (tín hiệu nhịp tim quang học)*

---

## Pipeline hệ thống

![Pipeline hệ thống](docs/images/System%20Pipeline.png)
*Pipeline hệ thống từ đầu đến cuối: từ tín hiệu cảm biến đến email báo cáo*

---

## Xử lý tín hiệu & AI

![DSP và AI Pipeline](docs/images/dsp_ai.png)
*Pipeline DSP và AI: xử lý nhịp tim PPG (trái) và phân loại đòn đấm IMU bằng Decision Tree (phải)*

**Tham số chính:**
- Activity gate: `acc_std < 1500 LSB` → bỏ qua cửa sổ
- Decision Tree: ngưỡng `acc_std = 2488.65` → Thường (0) hoặc Mạnh (1)
- Cross-talk guard: nếu `Acc_Mag > 14000 LSB` (7g), dừng cập nhật BPM
- I2C watchdog: tự khôi phục trong 20 ms, tối đa 3 lần thử

---

## Phần cứng

| Linh kiện | Vai trò | Cấu hình |
|-----------|---------|----------|
| Seeed Studio XIAO ESP32-S3 | Vi điều khiển chính | 240 MHz, 512 KB SRAM |
| MPU6050 | IMU — đo gia tốc | ±16g, I2C 0x68 |
| MAX30102 | PPG — đo nhịp tim quang học | IR 940 nm, I2C 0x57 |

![Phần cứng](docs/images/Hardware.png)
*Sơ đồ đấu dây: ESP32-S3 kết nối MPU6050 và MAX30102 qua bus I2C chung*

![Nguyên mẫu thực tế](docs/images/Prototype.png)
*Nguyên mẫu đeo trên cổ tay khi thử nghiệm*

Kết nối: SDA = GPIO 5, SCL = GPIO 6, 400 kHz, nguồn 3.3 V.

---

## Cài đặt

### 1. Cấu hình thông tin đăng nhập

Mở [`firmware/main.cpp`](firmware/main.cpp) và điền vào:

```cpp
#define WIFI_SSID       "tên_wifi"
#define WIFI_PASSWORD   "mật_khẩu_wifi"
#define AUTHOR_EMAIL    "gmail_của_bạn@gmail.com"
#define AUTHOR_PASSWORD "xxxx xxxx xxxx xxxx"   // Gmail App Password
#define RECIPIENT_EMAIL "gmail_nhận_báo_cáo@gmail.com"
```

> Thiết bị vẫn hoạt động ở **chế độ offline** nếu WiFi không kết nối được — chỉ bỏ qua bước gửi email.

### 2. Build và flash

```bash
# Yêu cầu: PlatformIO CLI hoặc VS Code + extension PlatformIO
pio run -e seeed_xiao_esp32s3 -t upload
pio device monitor   # 115200 baud
```

---

## Kết quả

Độ chính xác 89.9% (LOGO cross-validation) qua 10 test case:

| Hạng mục | Kết quả |
|----------|---------|
| Phát hiện hiệp đấm mạnh | PASSED |
| Hiệp thường — không nhạy cảm thái quá | PASSED |
| Activity gate tại 1500 LSB | PASSED |
| Ranh giới Decision Tree (acc_std 2400–2550) | PASSED |
| Chuyển động thường không bị phân loại là Intense | PASSED |
| Bất biến khi xoay cổ tay | PASSED |
| Đấm nhanh (3–4 cú/giây) | PASSED |
| Cross-talk guard khi va chạm mạnh | PASSED |
| FSM tự động reset sau 3 giây im lặng | PASSED |

**Video demo:** [Google Drive](https://drive.google.com/file/d/1EUahLwN11yc4-B6_D1cpn5PuJAG3qsEr/view?usp=sharing)

---

## Cấu trúc thư mục

```
├── firmware/
│   ├── main.cpp          # Firmware chính
│   └── classifier.h      # Decision Tree nhúng dưới dạng C header
├── data/
│   ├── raw/              # File CSV thô từ các hiệp thu thập
│   └── processed/        # 39k mẫu đã gán nhãn
├── notebooks/            # Training model + xuất classifier.h
├── scripts/              # Thu thập dữ liệu qua Serial
└── docs/
    ├── images/           # Hình ảnh minh hoạ cho README
    └── visual_qc/        # Đồ thị QC từng hiệp thu thập
```

---

## Phát triển tiếp theo

Dự án được tiếp tục với kiến trúc **FreeRTOS + BLE**, mở rộng từ phân loại đòn đấm sang nhận dạng 5 hoạt động thể chất (Đi bộ / Chạy / Ngồi / Đứng / Nằm).

→ [wearable-health-monitor](https://github.com/ChuoiUhuhu0727/wearable-health-monitor)
