# Các chương trình Arduino (Arduino Sketches)

Thư mục này chứa các chương trình (sketches) Arduino cho dự án Con lắc ngược quay.

## Yêu cầu trước

Cài đặt [arduino-cli](https://arduino.github.io/arduino-cli/latest/installation/) và core AVR:

```bash
arduino-cli core install arduino:avr
```

## Nạp chương trình

1. Tìm cổng serial của Arduino:
   ```bash
   arduino-cli board list
   ```

2. Biên dịch và nạp chương trình (từ thư mục gốc của repo):
   ```bash
   arduino-cli compile --upload -p <PORT> --fqbn arduino:avr:nano:cpu=atmega328 RotaryInvertedPendulum-arduino/<SketchName>
   ```

   **Lưu ý:** Dự án này sử dụng `cpu=atmega328` (bootloader mới). Không sử dụng `cpu=atmega328old`.

## Các chương trình (Sketches)

### Các chương trình kiểm tra (Test Sketches)

Các chương trình này dùng để kiểm tra hoạt động của từng linh kiện phần cứng riêng lẻ.

#### TestHeartbeat

Nhấp nháy LED theo chu kỳ xung kép. Hữu ích để xác minh Arduino đã được cấp nguồn và đang hoạt động.

- **Chu kỳ:** BẬT(100ms)-TẮT(100ms)-BẬT(100ms)-TẮT(1000ms), lặp lại
- **Serial (115200 baud):** Hiển thị số lượng nhịp đập (heartbeat count)

**Trường hợp sử dụng:** Xác minh nhanh phần cứng, kiểm tra nguồn điện, kiểm tra bootloader, debug cơ sở.

#### TestEncoder

Kiểm tra cảm biến mã hóa vòng quay từ tính AS5600 với thuật toán theo dõi nhiều vòng quay.

- **LED sáng** trong quá trình thiết lập, chờ phát hiện nam châm
- **LED nhấp nháy** khi đang xuất dữ liệu đọc được
- **Serial (115200 baud):** Xuất dữ liệu dưới dạng `pendulum_deg:value` (tương thích với Serial Plotter)

**Khắc phục sự cố:**
- "Waiting for magnet..." → Đảm bảo nam châm được đặt gần cảm biến AS5600
- "Magnet strength too weak/strong" → Điều chỉnh khoảng cách nam châm (~1-2mm)

#### TestMotor

Kiểm tra động cơ bước bằng cách quay qua lại giữa hai vị trí +90° và -90°.

- **LED sáng** trong quá trình thiết lập và khi động cơ chuyển động
- **Serial (115200 baud):** Hiển thị trạng thái chuyển động
- Tốc độ tối đa được đặt ở mức 20,000 steps/sec (chậm hơn 10 lần so với chế độ sản xuất để dễ quan sát)

#### TestSerial

Đo thời gian truyền thông khứ hồi (round-trip time - RTT) qua cổng serial bằng cách phản hồi các byte nhận được.

- **LED sáng** trong quá trình thiết lập
- **LED nhấp nháy** khi nhận được mỗi byte dữ liệu

Chạy tập lệnh đo lường bằng Julia:
```bash
julia --project=./RotaryInvertedPendulum-julia ./RotaryInvertedPendulum-arduino/TestSerial/measure_serial_rtt.jl <PORT> 115200
```

Kết quả kỳ vọng: RTT ~2.5ms, tần số tối đa lý thuyết đạt ~400 Hz.

#### TestServer

Một server đơn giản theo dõi các sóng sine/cosine và phản hồi các yêu cầu byte 'S' hoặc 'C'.

```bash
julia --project=./RotaryInvertedPendulum-julia ./RotaryInvertedPendulum-arduino/TestServer/client.jl
```

### Các chương trình sản xuất (Production Sketches)

#### LowLevelServer

Server cấp thấp cho hoạt động điều khiển từ máy tính. Arduino đóng vai trò là thiết bị tớ (slave), nhận lệnh qua cổng serial (tốc độ 2,000,000 baud) từ mã điều khiển Julia/Python chạy trên máy tính.

```bash
julia --project=./RotaryInvertedPendulum-julia ./RotaryInvertedPendulum-arduino/LowLevelServer/client.jl --visualise
```

#### PIDControl

Bộ điều khiển PID độc lập chạy hoàn toàn trên Arduino. Không cần máy tính sau khi đã nạp chương trình.

- **Serial (500000 baud):** Kết nối để xem chẩn đoán và thu thập dữ liệu
- **Tần suất nháy LED:** Nhanh (100ms) = đang chờ, Chậm (500ms) = đã kích hoạt xuất dữ liệu

**Các lệnh Serial:**
- `P` - Bật/tắt xuất dữ liệu (định dạng CSV ở tần số 100 Hz)
- `M` - Hiển thị trạng thái nam châm
- `R` - Reset lại trạng thái PID

**Thu thập dữ liệu:**
```bash
julia --project=./RotaryInvertedPendulum-julia ./RotaryInvertedPendulum-arduino/PIDControl/collect_and_plot.jl <PORT> [DURATION]
```

Ví dụ:
```bash
julia --project=./RotaryInvertedPendulum-julia ./RotaryInvertedPendulum-arduino/PIDControl/collect_and_plot.jl /dev/cu.usbserial-10 10
```

Lệnh này sẽ thu thập dữ liệu trong khoảng thời gian chỉ định, tạo các đồ thị và lưu tệp CSV vào thư mục `PIDControl/experiments/`.

Xem tệp `PIDControl/TUNING_HISTORY.md` để biết các ghi chú tinh chỉnh và phần tiêu đề của `PIDControl/PIDControl.ino` để biết chi tiết kiến trúc.

## Kiểm tra phần cứng trong vòng lặp (Hardware-in-the-Loop Testing)

Để chạy kiểm tra tự động khi đã kết nối phần cứng:

```bash
./RotaryInvertedPendulum-arduino/scripts/monitor_serial.sh <PORT> <BAUD> <DURATION>
```

Ví dụ:
```bash
./RotaryInvertedPendulum-arduino/scripts/monitor_serial.sh /dev/cu.usbserial-10 115200 10
```
