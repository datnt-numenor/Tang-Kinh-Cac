# Hướng dẫn: Chuyển từ Mock Drone sang ESP32 thật

## Tổng quan

Hệ thống hiện tại dùng 12 script Python (`tools/mock_drone.py`) chạy trong Docker để giả lập drone. Khi chuyển sang ESP32 thật, bạn **KHÔNG CẦN SỬA** bất kỳ code nào ở Backend, Frontend hay MQTT Listener. Chỉ cần:

1. Sửa 1 dòng config WiFi/IP trên ESP32
2. Comment bỏ mock trong `docker-compose.yml`
3. Nạp firmware vào ESP32, cấp nguồn

Dưới đây là các bước chi tiết.

---

## Bước 1: Tìm IP máy tính chạy Docker

ESP32 cần biết IP của máy tính để kết nối tới MQTT Broker (chạy trong Docker, đã expose port `1883`).

```powershell
# Windows
ipconfig
# Tìm dòng "IPv4 Address" của adapter WiFi đang dùng
# Ví dụ: 192.168.1.15
```

> **Yêu cầu:** ESP32 và máy tính **phải cùng mạng WiFi** (cùng router/hotspot).

---

## Bước 2: Sửa cấu hình trên ESP32

Mở file `esp32-firmware/main.py`, sửa **3 dòng** ở đầu file:

```python
# --- 1. CẤU HÌNH HỆ THỐNG ---
WIFI_SSID = "TenWiFiCuaBan"          # ← Sửa thành tên WiFi đang dùng
WIFI_PASS = "MatKhauWiFi"            # ← Sửa thành mật khẩu WiFi
MQTT_SERVER = "192.168.1.15"         # ← Sửa thành IP máy tính (Bước 1)
```

> **Không cần sửa gì khác.** `CLIENT_ID` đã tự lấy từ MAC Address (`machine.unique_id()`), các topic MQTT (`drone/telemetry`, `drone/{id}/command`) và format JSON đã khớp 100% với Backend.

---

## Bước 3: Tắt Mock Drone trong Docker

Mở file `docker-compose.yml`, comment (hoặc xóa) tất cả các service `mock_1` đến `mock_12`:

```yaml
# mock_1:
#   build: ./backend-app
#   environment:
#     - MQTT_HOST=mqtt_broker
#   command: python /tools/mock_drone.py --id mock_1
#   volumes:
#     - ./tools:/tools
#   depends_on:
#     - mqtt_broker
#   restart: always

# ... tương tự cho mock_2 đến mock_12
```

Sau đó chạy lại Docker:

```powershell
docker-compose up -d --remove-orphans
```

Lệnh `--remove-orphans` sẽ tự dọn sạch 12 container mock đã bị xóa khỏi file.

> **Giữ lại các service:** `mqtt_broker`, `redis`, `backend`, `mqtt_listener`, `frontend` — đây là hệ thống chính.

---

## Bước 4: Nạp firmware vào ESP32

Dùng Thonny IDE (hoặc ampy/mpremote):

1. Kết nối ESP32 với máy tính qua USB.
2. Upload **3 file** vào bộ nhớ gốc (`/`) của ESP32:
   - `esp32-firmware/boot.py`
   - `esp32-firmware/mqtt.py`
   - `esp32-firmware/main.py`
3. Nhấn nút Reset trên ESP32 (hoặc rút USB cắm lại).

---

## Bước 5: Kiểm tra kết nối

### 5.1. Xem Serial Monitor (Thonny)

Sau khi reset, ESP32 sẽ in ra:

```
Đang nối WiFi: TenWiFiCuaBan
...
WiFi OK! IP: 192.168.1.20
✅ ĐÃ KẾT NỐI! Đang lắng nghe lệnh tại: drone/a4cf12f8e3b0/command
📡 [IDLE] Lat: 20.98081, Lng: 105.79593, Alt: 0.0m
```

Nếu thấy `WiFi lỗi!` → kiểm tra lại SSID/Password.  
Nếu thấy `Lỗi MQTT` → kiểm tra lại IP và đảm bảo Docker đang chạy.

### 5.2. Xem trên giao diện Web

1. Mở `http://localhost:5173` (hoặc `http://<IP_máy_tính>:5173` từ điện thoại).
2. Vào tab **Discovery** → ESP32 sẽ xuất hiện trong danh sách "Drone mới phát hiện" với tên `Drone a4cf12` (6 ký tự đầu của MAC).
3. Ấn **Nhận Drone** → đặt tên (VD: `Drone-Thật-01`).
4. Chuyển sang tab **Map** → bạn sẽ thấy vị trí tọa độ giả lập (vì ESP32 hiện gửi tọa độ cố định `20.980812, 105.795931`).

### 5.3. Test gửi lệnh từ Web

1. Trên giao diện Map, chọn drone vừa claim.
2. Gửi lệnh **Takeoff** → Trên Serial Monitor sẽ thấy:
   ```
   Lệnh mới nhận được: TAKEOFF
   ```
3. Dùng chức năng **Lên lịch đội hình** → Thực thi → Serial sẽ hiện:
   ```
   Lệnh mới nhận được: GOTO
   🚩 Đang bay tới: 20.981, 105.7962
   ```

---

## Bước 6: Kết nối cảm biến thật (Khi có phần cứng)

Hiện tại `main.py` đang dùng **tọa độ giả lập** (`lat, lng = 20.980812, 105.795931`). Khi gắn cảm biến GPS thật (NEO-6M), bạn thay thế phần đọc tọa độ:

```python
# Trước (giả lập):
lat, lng, alt = 20.980812, 105.795931, 0.0

# Sau (đọc GPS thật):
from machine import UART
uart_gps = UART(2, baudrate=9600, tx=17, rx=16)

def read_gps():
    """Đọc tọa độ từ module GPS NEO-6M qua UART"""
    global lat, lng
    if uart_gps.any():
        line = uart_gps.readline()
        if line and b'$GPGGA' in line:
            parts = line.decode().split(',')
            # Parse NMEA format → decimal degrees
            # ... (tùy thư viện micropyGPS)
```

> **Quan trọng:** Khi có GPS thật, bạn cũng cần sửa logic di chuyển (GOTO) — thay vì cộng trừ `step` trên biến `lat`/`lng` (giả lập), hãy đọc tọa độ thật từ GPS và so sánh với `target_lat`, `target_lng` để biết drone đã tới đích chưa.

---

## Tóm tắt: Cái gì cần sửa, cái gì KHÔNG

| Thành phần | Cần sửa? | Chi tiết |
|:---|:---:|:---|
| `esp32-firmware/main.py` | ✅ | Sửa 3 dòng: `WIFI_SSID`, `WIFI_PASS`, `MQTT_SERVER` |
| `docker-compose.yml` | ✅ | Comment/xóa `mock_1` → `mock_12` |
| `backend-app/` (toàn bộ) | ❌ | Không sửa gì — API, WebSocket, DB đều tương thích |
| `frontend-app/` (toàn bộ) | ❌ | Không sửa gì — Map, Lên lịch, Discovery đều hoạt động |
| `mqtt_listener.py` | ❌ | Không sửa — đã nhận đúng format JSON từ ESP32 |
| `infrastructure/mosquitto.conf` | ❌ | Không sửa — đã cho phép anonymous, đã expose port 1883 |
