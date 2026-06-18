# Nhật ký tinh chỉnh PID (PID Tuning History)

Tài liệu này theo dõi quá trình tinh chỉnh từng bước (iterative tuning) cho bộ điều khiển cân bằng PID.

## Thiết lập phần cứng

- **Động cơ:** Động cơ bước NEMA17 với driver DRV8825, chế độ vi bước 8x (1600 bước/vòng - steps/rev)
- **Cảm biến mã hóa vòng quay:** AS5600 magnetic encoder (12-bit, 4096 xung/vòng - counts/rev)
- **Bộ điều khiển:** Arduino Nano (ATmega328P, 16MHz)
- **Giới hạn động cơ:** ±90° từ vị trí bắt đầu (ràng buộc dây nối, không có cổ góp slip ring)

## Kiến trúc điều khiển

- Giản đồ trạng thái (State machine): WAITING (Chờ) → BALANCING (Cân bằng, kích hoạt trong khoảng ±25° so với phương thẳng đứng)
- Đầu ra PID điều khiển vị trí động cơ (không phải vận tốc)
- Lọc thông thấp góc lệch con lắc trước khi tính toán PID
- Chống bão hòa tích phân (Anti-windup) cho thành phần tích phân với các giới hạn và suy giảm tại giới hạn động cơ

---

## Lần thử 1: Các giá trị Ziegler-Nichols ban đầu

**Ngày:** 10/01/2025

**Các tham số:**
| Tham số | Giá trị |
|-----------|-------|
| Kp | 1.2 (0.6 × Ku, với Ku=2.0) |
| Ki | 24.0 (2 × Kp / Tu, với Tu=0.1) |
| Kd | 0.015 (Kp × Tu / 8) |
| Bộ lọc (Filter) | 500 Hz |
| Tốc độ tối đa động cơ (Motor MaxSpeed) | 200000 |
| Gia tốc động cơ (Motor Accel) | 100000 |

**Các lỗi phát hiện khi review mã nguồn:**
- Lỗi chia số nguyên: `controlPeriod = 1/1000 = 0` (điều kiện luôn đúng)
- Không có chống bão hòa tích phân (integral anti-windup)
- Giới hạn động cơ chỉ dừng cứng thay vì cắt giới hạn (clamping)
- Hàm Tare (hiệu chuẩn góc) không hoạt động

**Quan sát:**
- Động cơ bị trôi mạnh về phía các giới hạn ±90°
- Đạt ngưỡng bão hòa và mất cân bằng
- Cân bằng được khoảng ~6 giây trước khi rơi
- Hiện tượng bão hòa tích phân (integral wind-up) gây ra trôi lệch liên tục

---

## Lần thử 2: Giảm Ki, Tăng Kd

**Ngày:** 10/01/2025

**Các tham số:**
| Tham số | Giá trị | Thay đổi |
|-----------|-------|--------|
| Kp | 1.2 | - |
| Ki | 6.0 | ↓ từ 24.0 |
| Kd | 0.03 | ↑ từ 0.015 |
| Bộ lọc | 500 Hz | - |

**Quan sát:**
- Động cơ không còn bị bão hòa ở giới hạn ±90° (nằm trong khoảng ±60°)
- Động cơ quay trở lại tâm sau khi con lắc rơi
- Vẫn dao động rất mạnh trong các nỗ lực cân bằng
- Các dao động nhanh ~±20-30° không bị triệt tiêu
- Hệ thống ở ranh giới của sự ổn định

**Chẩn đoán:** Thiếu giảm chấn (damping), thành phần vi phân (derivative) khuếch đại nhiễu tần số cao.

---

## Lần thử 3: Tăng giảm chấn, Hạ tần số cắt bộ lọc

**Ngày:** 10/01/2025

**Các tham số:**
| Tham số | Giá trị | Thay đổi |
|-----------|-------|--------|
| Kp | 1.0 | ↓ từ 1.2 |
| Ki | 6.0 | - |
| Kd | 0.08 | ↑ từ 0.03 |
| Bộ lọc | 100 Hz | ↓ từ 500 Hz |

**Quan sát:**
- Các dao động tần số cao bị loại bỏ (bộ lọc hoạt động tốt)
- Động cơ nằm trong khoảng ±60° (không bị bão hòa)
- Khoảng thời gian ổn định tốt nhất đạt được ở cuối lượt chạy với sai số ~±10-15°
- Vẫn bị rơi và cần phục hồi nhiều lần
- Các dao động chậm hơn vẫn tồn tại khi cân bằng
- Cải thiện rõ rệt so với các lần thử trước

**Chẩn đoán:** Tốt hơn, nhưng vẫn cần nhiều giảm chấn hơn hoặc giảm bớt độ nhạy điều khiển.

---

## Lần thử 4: Sửa lỗi + Cải tiến tính năng chẩn đoán

**Ngày:** 10/01/2025

**Các lỗi đã sửa:**
- Sửa lỗi thời gian vòng lặp đầu tiên: `prev_time_us` hiện được khởi tạo trong hàm `setup()` để tránh giá trị dt khổng lồ ở lượt chạy đầu tiên
- Lỗi này từng gây ra các đột biến vi phân và tính toán sai bộ lọc khi khởi động

**Cải tiến thu thập dữ liệu:**
- Thu thập riêng lẻ các thành phần PID (P, I, D) để chẩn đoán
- Thêm trạng thái (WAITING/BALANCING) vào đầu ra
- Đồ thị mới: `plot_pid_terms.png` hiển thị từng thành phần theo thời gian

**Các tham số:** Giống như Lần thử 3 (Kp=1.0, Ki=6.0, Kd=0.08, Bộ lọc=100Hz)

**Tiếp theo:** Chạy với các lỗi đã sửa để thiết lập baseline mới, sau đó tiếp tục tinh chỉnh.

---

## Lần thử 5: Tối ưu hóa thời gian & Vòng lặp 1 kHz cố định

**Ngày:** 10/01/2025

**Vấn đề:** Thời gian vòng lặp không nhất quán, thay đổi nhiều dựa trên mã nguồn thực thi ở mỗi lượt. Đầu ra serial gây ra độ trễ đáng kể và cảnh báo quá thời gian (overrun) giả.

**Các thay đổi:**

1. **Vòng lặp điều khiển tần số cố định (1 kHz)**
   - Triển khai mẫu thiết kế thoát sớm (early-return): vòng lặp thoát ngay lập tức nếu chưa trôi qua <1000μs
   - Đảm bảo dt nhất quán cho các tính toán PID
   - Thêm tính năng phát hiện quá thời gian (đánh dấu các lượt chạy >1.5 lần chu kỳ kỳ vọng)

2. **Tăng tốc độ I2C (100 kHz → 400 kHz)**
   - Thời gian đọc encoder AS5600 giảm từ ~650μs xuống ~290μs
   - Đây là điểm nghẽn chính chiếm 65% thời gian vòng lặp

3. **Tối ưu hóa đầu ra Serial**
   - Vấn đề: Hàm `Serial.print(float, 2)` mất ~500μs cho mỗi cuộc gọi do định dạng số thực float bằng phần mềm trên AVR
   - Giải pháp: Truyền các giá trị dưới dạng số nguyên (nhân 1000), giải mã trong Julia
   - Tối ưu hóa thêm: Sử dụng hàm `ltoa()` + nối chuỗi thủ công vào bộ đệm thay vì dùng `snprintf`
   - Kết quả: Đầu ra serial không còn gây ra lỗi quá thời gian

4. **Tăng tốc độ baud (115200 → 500000)**
   - Giảm thời gian chờ đợi bộ đệm TX
   - Arduino Nano hỗ trợ tối đa 2 Mbaud, nhưng mức 500k hoạt động ổn định

**So sánh đầu ra Serial:**

| Phương pháp | Lượt quá thời gian (Overruns) | Tần số vòng lặp | Kích thước Flash |
|----------|----------|-----------|------------|
| Float Serial.print() ×9 cuộc gọi | 1065 | 881 Hz | 12,994 B |
| Số nguyên ×1000 Serial.print() ×9 | 0 | 985 Hz | 12,560 B |
| Bộ đệm snprintf số nguyên | 0 | 1001 Hz | 13,904 B |
| Bộ đệm ltoa số nguyên (cuối cùng) | 0 | 1003 Hz | 12,700 B |

**Kết quả thời gian cuối cùng:**
- Tần số vòng lặp: trung bình 1003 Hz (mục tiêu: 1000 Hz)
- Phạm vi tần số vòng lặp: 900-1100 Hz
- Lượt quá thời gian: 0

**Các tham số:** Kp=0.8, Ki=4.0, Kd=0.015, Bộ lọc=100Hz

---

## Các bước tiếp theo

Hạ tầng thời gian hiện tại đã ổn định. Sẵn sàng tập trung vào tinh chỉnh PID với khả năng thu thập dữ liệu đáng tin cậy.

---

## Phương pháp tinh chỉnh hệ thống (Systematic Tuning Approach)

1. **Sửa lỗi trước** - đảm bảo hệ thống hoạt động như mong đợi
2. **Thu thập dữ liệu chẩn đoán** - các thành phần PID, trạng thái, thời gian
3. **Thay đổi từng tham số một** - cô lập ảnh hưởng của mỗi thay đổi
4. **So sánh định lượng** - sử dụng sai số RMS, các chỉ số thời gian cân bằng
5. **Tài liệu hóa mọi thứ** - theo dõi những gì hiệu quả và những gì không

---

## Ghi chú

- Thư viện AccelStepper trên Arduino Nano bị giới hạn ở khoảng ~4000 steps/sec khi sử dụng hàm `run()`
- Các thiết lập maxSpeed/acceleration hiện tại (200000/100000) vượt quá giới hạn này nhưng gia tốc đủ cao để hoạt động gần như tức thời
- Biên độ kích hoạt ±25° hoạt động phù hợp
- Giới hạn động cơ ±90° là một ràng buộc cứng do dây nối
