# ESP32S3_Quadcopter_FlightController
# ARDUINO IDE (OUTLOG.MD)
**Dự án**: Quadcopter Flight Controller ESP32-S3 N16R8  
**Cảm biến & Ngoại vi**: MPU6500 (GY-6500), GPS NEO-6M, 4x ESC 2100KV Brushless, 8MB Octal PSRAM, 16MB Flash  
**Cập nhật mới**: Waypoint Home RTH, Failsafe Mất Sóng & Pin Yếu, Cứu Hộ 3-Motor, Giới Hạn Ga Căn Chỉnh 50-60%, Dual RC Virtual Joysticks.  

---

## PHẦN 1: TỔNG HỢP CÁC TÍNH NĂNG ĐÃ NÂNG CẤP HOÀN THIỆN (100%)

### 1. Tay Cầm Điều Khiển 2 Cần Joystick Ảo (RC Mode 2) Trên Web UI
([`web_portal.cpp`](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Quadcopter_FlightController/src/telemetry/web_portal.cpp))
- [x] **Cần Trái (Left Stick - Ga & Xoay)**:
  - Trục dọc (Y): Điều khiển Ga (Throttle) từ 1000us đến 2000us (Cao / Thấp).
  - Trục ngang (X): Điều khiển Xoay mũi (Yaw rate) từ $-150^\circ/\text{s}$ đến $+150^\circ/\text{s}$ (Trái / Phải), tự động hồi vị về giữa (Spring return) khi thả tay.
- [x] **Cần Phải (Right Stick - Tiến/Lùi & Nghiêng)**:
  - Trục dọc (Y): Điều khiển Tiến / Lùi (Pitch $-25^\circ \to +25^\circ$).
  - Trục ngang (X): Điều khiển Nghiêng Trái / Phải (Roll $-25^\circ \to +25^\circ$).
  - Cả 2 trục tự động lò xo hồi tâm về $0^\circ$ (tự động giữ thăng bằng cân bằng khi buông tay).
- [x] **Cảm ứng đa điểm (Multi-touch)**:
  - Hai ngón cái có thể điều khiển đồng thời cả 2 cần joystick mượt mà trên màn hình điện thoại.
- [x] **Nút Khởi động (ARM / DISARM)**: Nút bật/tắt động cơ an toàn.
- [x] **Nút Về Nhà (RETURN TO HOME - RTH)**: Nút kích hoạt bay về điểm Home bằng 1 chạm.

---

### 2. Tự Động Về Nhà (Auto Return-To-Home & Waypoint Nhà)
([`main.cpp`](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Quadcopter_FlightController/src/main.cpp) & [`gps_neo6m_drv.cpp`](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Quadcopter_FlightController/src/drivers/gps_neo6m_drv.cpp))
- [x] **Tự động khóa tọa độ Waypoint Nhà (Home Point)**:
  - Khi GPS NEO-6M đạt 3D Fix với $\ge 6$ vệ tinh, tọa độ cất cánh (Lat/Lon/Alt) được tự động khóa làm Home.
- [x] **Kích hoạt tự động khi Mất Sóng (Signal Lost Watchdog)**:
  - Nếu mất kết nối Wi-Fi / RC quá **2.0 giây**, hệ thống tự động kích hoạt chế độ `MODE_RETURN_TO_HOME`.
- [x] **Kích hoạt tự động khi Pin Yếu (Low Battery Failsafe)**:
  - Cảnh báo khi điện áp $< 7.0$V (3.5V/cell).
  - Tự động kích hoạt RTH bay về khi điện áp chạm ngưỡng nguy hiểm $< 6.6$V (3.3V/cell).
- [x] **Thuật toán tự dẫn đường bay về Home**:
  - Tự động tính toán góc phương vị (Bearing) tới Home.
  - Tự quay mũi máy bay hướng về Home (Yaw PID error $\to$ rate).
  - Tự duy trì góc nghiêng tiến nhẹ (Pitch $-6^\circ$) với ga bay hành trình 1450us.
  - Khi tới đỉnh điểm Home (khoảng cách $< 3$ mét) $\to$ Tự động chuyển sang `MODE_FAILSAFE_LAND` (từ từ giảm ga tiếp đất êm dịu và tự ngắt động cơ).

---

### 3. Cứu Hộ Khi 1 Trong 4 Motor Bị Mất Kết Nối / Hỏng (Fault-Tolerant Degraded Control)
([`pid_mixer.cpp`](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Quadcopter_FlightController/src/control/pid_mixer.cpp))
- [x] **Thuật toán tự động phát hiện động cơ chết (`detect_motor_failure`)**:
  - Nếu một động cơ đang được lệnh cấp ga cao ($>1450$us) nhưng con quay hồi chuyển phát hiện gia tốc xoay bị lật về góc của động cơ đó liên tục $> 200$ms $\to$ Hệ thống lập tức cô lập motor đó và kích hoạt chế độ cứu nạn `MODE_MOTOR_FAIL_DESCENT`.
- [x] **Bộ trộn lực quay rơi có kiểm soát 3 động cơ (Controlled Spin Descent)**:
  - Thay vì cố cưỡng bức cân bằng Yaw khiến máy bay bị lật nhào rơi tự do, hệ thống hy sinh trục xoay Yaw (cho phép máy bay tự xoay tròn nhanh theo trục đứng như hạt cây chò).
  - Motor đối diện được hãm xuống ~25% ga để triệt tiêu mô-men bẻ gãy.
  - Cặp 2 motor liền kề gánh toàn bộ lực nâng chính để hãm tốc độ rơi.
  - Áp dụng giảm chấn Gyro Damping trên trục Roll/Pitch giữ đĩa quay luôn phẳng $\to$ Máy bay hạ cánh an toàn từ từ chạm đất mà không bị lộn nhào hay gãy cánh quạt!

---

### 4. Giới Hạn Ga Cân Bằng Động Cơ An Toàn 50% - 60% Ga (Chống Quá Tải / Cháy Motor)
([`auto_calibrator.cpp`](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Quadcopter_FlightController/src/calibration/auto_calibrator.cpp))
- [x] **Giai đoạn 2 (Test Rung & Deadband)**:
  - Điều chỉnh mốc test công suất tối đa xuống **1550us (55%–60% ga)**, loại bỏ hoàn toàn mốc 100% ga nguy hiểm.
- [x] **Giai đoạn 3 (Chạy Thử Tải Cân Bằng Lực Đẩy Động)**:
  - Tăng dốc ga từ 1100us (10%) lên tối đa **1550us (55%–60% ga)**.
  - Thuật toán thích nghi $K_{M1..4}$ lấy mẫu phân tích ở dải ga an toàn 1400us – 1550us.
  - Đảm bảo ESC 5A–8A chạy mát, không bị quá dòng, pin LiPo 2S không bị sụt áp đột ngột và motor 2100KV hoạt động bền bỉ!

---

## PHẦN 2: HƯỚNG DẪN NẠP ARDUINO IDE

Mã nguồn hoàn toàn tương thích và đã được cập nhật đồng bộ trong thư mục:  
📁 [**`ESP32S3_Drone_Arduino/`**](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Drone_Arduino/) $\to$ [**`ESP32S3_Drone_Arduino.ino`**](file:///storage/emulated/0/ESP32-S3%20DRONE/ESP32S3_Drone_Arduino/ESP32S3_Drone_Arduino.ino)

### Thiết Lập Menu `Tools` Trong Arduino IDE:
1. **Board**: `ESP32S3 Dev Module`
2. **USB CDC On Boot**: `Enabled`
3. **CPU Frequency**: `240MHz (WiFi)`
4. **Flash Mode**: `QIO 80MHz`
5. **Flash Size**: `16MB (128Mb)`
6. **Partition Scheme**: `16M Flash (3MB APP/9.9MB FATFS)`
7. **PSRAM**: **`OPI PSRAM`** *(Bắt buộc chọn OPI)*
8. Cài 2 thư viện: `TinyGPSPlus` và `WebSockets`.
9. Bấm **Upload** để nạp trực tiếp vào ESP32-S3!

*(Hoặc dùng script nạp nhanh 1 chạm: [`binaries_for_flashing/flash_script.sh`](file:///storage/emulated/0/ESP32-S3%20DRONE/binaries_for_flashing/flash_script.sh))*
