# Ghi chú thiết kế mạch điện tử

Tại sao mỗi linh kiện trong danh sách [BOM](BOM.md) được lựa chọn, và những điều cần biết khi tìm nguồn cung cấp thay thế hoặc thay thế linh kiện. Tài liệu BOM là tài liệu tham khảo mua sắm; tài liệu này giải thích lý do thiết kế đằng sau nó.

## Sơ đồ đấu dây

<img src="../diagrams/system-without-batteries.jpg" height="600">

Tất cả các linh kiện được bố trí trên một bo mạch lỗ (protoboard) duy nhất kích thước 40 × 60 mm. Sơ đồ trên là sơ đồ bố trí chuẩn; ảnh thực tế của các linh kiện có trong thư mục [`../diagrams/`](../diagrams/).

## Vi điều khiển — Arduino Nano (ATmega328P, 16 MHz)

- Bộ nhớ 32 KB flash / 2 KB SRAM là đủ cho [giao thức nhị phân LowLevelServer](../RotaryInvertedPendulum-arduino/LowLevelServer/LowLevelServer.ino) cùng với các cổng I/O cho cảm biến AS5600 và thư viện AccelStepper chạy ở tần số bước nội bộ 1 kHz. Phiên bản PID độc lập chạy trên thiết bị cũng vừa vặn và còn dư bộ nhớ.
- Các tác vụ nặng (suy luận RL - RL inference, MPC, sysid) được thực hiện trên máy tính (host PC). Board Nano chỉ đóng vai trò chuyển tiếp trạng thái và lệnh điều khiển ở tốc độ baud 2,000,000 (2 Mbaud) — nó không cần sức mạnh xử lý của các MCU cao cấp.
- Phiên bản USB-C được ưu tiên hơn Mini/Micro vì độ bền đầu nối tốt hơn — thiết bị này thường xuyên phải cắm và rút cáp trong quá trình phát triển nạp firmware.
- Bất kỳ board clone nào sử dụng chip giao tiếp CH340 đều hoạt động được; driver đã được tích hợp sẵn trong macOS / Linux hiện đại.

## Động cơ bước — NEMA17 17HS4023 (Dòng định mức 1 A, thân dài 22 mm)

- **Được thiết kế chạy dưới tải định mức.** Cụm cánh tay + con lắc nặng dưới 50 g và quán tính quay duy nhất mà động cơ phải chống lại là chính cánh tay quay (khoảng ~1.5 × 10⁻⁵ kg·m²). Dòng điện chạy qua cuộn dây động cơ hiếm khi vượt quá ~0.3 A. Dòng định mức 1 A của động cơ cung cấp một khoảng dự phòng khoảng ~3 lần để tránh bị trượt bước (stall).
- **Chọn phiên bản thân ngắn 17HS4023 thay vì 17HS4401 thân dài hơn.** Động cơ được bắt vít thẳng đứng, do đó khối lượng của nó nằm dưới trục quay và không gây tải lên vòng bi — nhưng thân ngắn hơn vẫn giúp giảm chiều cao thiết bị và tiết kiệm chi phí. Với mức tải của thiết bị này, mô-men xoắn bổ sung của động cơ dài hơn là lãng phí.
- Việc thay thế bằng một động cơ nặng hơn hoặc mạnh hơn sẽ làm tăng khối lượng và chi phí mà không mang lại lợi ích gì; việc thay thế bằng động cơ nhỏ hơn (ví dụ: NEMA14) có nguy cơ bị mất bước dưới các lệnh swing-up đột ngột.

## Driver động cơ bước — DRV8825

- **Vref được đặt thành 0.485 V → giới hạn dòng điện ~0.9 A** mỗi pha (bằng 90% dòng định mức 1 A của động cơ). Khoảng dự phòng 10% tiêu chuẩn này giúp giữ cho driver và động cơ luôn dưới giới hạn nhiệt độ hoạt động an toàn vô thời hạn.
- Dải điện áp cấp từ 8.2–45 V; 12 V được chọn là điện áp thấp nhất hợp lý — xem phần "Nguồn điện" bên dưới.
- **A4988** là một giải pháp thay thế tương thích chân (drop-in) nhưng đạt giới hạn ở dòng điện thấp hơn và phát ra tiếng ồn lớn hơn.
- **TMC2209** sẽ chạy êm hơn thông qua nội suy vi bước bên trong nhưng làm tăng độ phức tạp của cấu hình UART cho một lợi ích không đáng kể trên thiết bị này: đầu ra vi bước 8× (8× microstepping) từ AccelStepper đã đủ mượt mà ở tốc độ hoạt động của chúng ta.
- **Hãy đặt Vref trước khi kết nối động cơ.** Khi driver đã được cấp nguồn và động cơ chưa được kết nối, hãy đo Vref so với GND trong khi xoay biến trở tinh chỉnh (trim pot).

### Giá trị Vref cho các driver thay thế

Nếu bạn chuyển sang sử dụng A4988 hoặc TMC2209, quy trình điều chỉnh Vref là tương tự nhưng mối quan hệ giữa Vref và giới hạn dòng điện pha kết quả sẽ khác nhau:

| Driver | Công thức tính Imax → Vref | Giá trị Vref cho mục tiêu dòng 0.9 A |
| ------- | --------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| DRV8825 | `Vref = Imax / 2` (điện trở cảm dòng Rcs = 0.1 Ω, tiêu chuẩn trên Pololu và hầu hết các bản clone) | **0.45 V** (chúng tôi chạy ở mức 0.485 V — đủ gần) |
| A4988 | `Vref = Imax × 8 × Rcs`. Mạch Pololu sử dụng Rcs = 0.05 Ω; một số mạch clone sử dụng 0.1 Ω — hãy kiểm tra board của bạn | **0.36 V** (Pololu) / **0.72 V** (mạch clone có Rcs = 0.1 Ω) |
| TMC2209 | Việc tính toán dòng RMS khá phức tạp — hãy sử dụng công cụ [TMC220X Vref calculator](https://printpractical.github.io/VrefCalculator/) | Theo công cụ tính toán |

## Nguồn điện — 12 V, 2 A

Đây là nguồn điện lý tưởng cho thiết bị này. Lý do:

Driver DRV8825 băm dòng điện trong cuộn dây, do đó dòng điện cấp từ nguồn nguồn ≠ dòng điện pha của động cơ. Cân bằng công suất:

```
I_source ≈ (I_phase × V_coil) / V_source
         ≈ (0.9 A × ~3.5 V) / 12 V ≈ 0.26 A mỗi pha
```

Khi cả hai pha đều hoạt động cộng với ~50 mA cho mạch logic (Nano + AS5600 + đèn chỉ báo) sẽ tiêu thụ **khoảng ~0.6 A ở trạng thái ổn định (steady-state)**, với các đỉnh ngắn hạn lên tới ~1 A khi đảo chiều động cơ đột ngột. Nguồn 2 A cung cấp khoảng dự phòng ~3 lần — đây là biên độ an toàn phù hợp cho một adapter cắm tường giá rẻ mà không bị lãng phí công suất.

Các bộ nguồn 3 A và 5 A hoạt động tốt (đã được xác thực thực nghiệm) nhưng dòng điện dư thừa sẽ không được sử dụng. Điều thực sự quan trọng hơn thông số dòng điện lý thuyết là:

- **Chất lượng ổn áp (Regulation quality)** — một bộ nguồn 2 A sạch (ít nhiễu) tốt hơn một bộ nguồn 5 A bị nhiễu sóng hài (ripple). Mạch hạ áp LDO 5 V của Nano và bus I²C của AS5600 dễ bị lỗi khi đường nguồn bị nhiễu hơn là khi nguồn có công suất thấp.
- **Tụ lọc nguồn dung lượng lớn trên bo mạch (Bulk decoupling)** — tụ 22 µF trên đường nguồn xử lý hầu hết các xung nhiễu do băm dòng. Nếu bạn thấy hiện tượng sụt nguồn (brown-out) khi đảo chiều động cơ, hãy thêm một tụ 470 µF gần driver trước khi nghĩ đến việc nâng cấp bộ nguồn lớn hơn.
- **Tiếp xúc đầu nối (Connector contact)** — một jack nguồn tròn 5.5 mm bị lỏng sẽ gây sụt áp khi tải tăng đột ngột bất kể công suất nguồn lớn thế nào.

**Tại sao chọn cụ thể điện áp 12 V**:
- DRV8825 chấp nhận điện áp 8.2–45 V; 12 V là lựa chọn hợp lý và có chi phí rẻ nhất.
- IC ổn áp tuyến tính trên board của Arduino Nano hạ áp từ 12 V → 5 V một cách an toàn. Điện áp 24 V sẽ bắt đầu làm nó quá nóng (ổn áp chạy rất nóng và giảm hiệu năng trên mức ~16 V liên tục).
- Các adapter nguồn 12 V đầu cắm tròn 5.5 mm rất phổ biến và dễ tìm.

## Cảm biến mã hóa vòng quay từ tính — AS5600

- **Góc tuyệt đối 12-bit** → độ phân giải 2π / 4096 rad ≈ 0.088°. Sự lượng tử hóa này được mô hình hóa trong [`pendulum_env.py`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py) (`PENDULUM_LSB_RAD`) để chính sách quan sát cùng một kích cỡ bước nhảy trong cả mô phỏng và thực tế.
- **Không tiếp xúc / từ tính** → ma sát bằng không tại khớp con lắc, đây là bậc tự do (DOF) cơ học mà chúng ta cần bảo toàn nhất. Một cảm biến biến trở tiếp xúc hoặc đĩa quang mã hóa vòng quay sẽ thêm lực cản ma sát mà chúng ta phải nhận dạng và xáo trộn đối phó.
- Giao tiếp I²C ở tốc độ 400 kHz hoàn thành lượt đọc trong <1 ms — nằm gọn trong quỹ thời gian điều khiển.
- Các module AliExpress kiểu TZT đi kèm với một đĩa nam châm phân cực hướng kính nhỏ; không cần tìm nguồn nam châm riêng.
- **Căn chỉnh nam châm rất quan trọng.** Mặt đĩa nam châm cách mặt chip từ 0.5–3 mm, căn chỉnh đồng trục. Thanh ghi `AGC` (tự động kiểm soát độ lợi) của AS5600 báo cáo cường độ nam châm — hãy kiểm tra nó khi cấp nguồn lần đầu tiên; các giá trị ngoài phạm vi cho thấy nam châm bị lệch trục hoặc sai loại nam châm.

## Tụ lọc nguồn — 100 nF ceramic + 22 µF hoá

Lọc nguồn hai cấp trên đường nguồn 12 V tại chân VMOT của driver:

- **Tụ gốm 100 nF (104)** xử lý các xung nhiễu tần số cao từ quá trình băm dòng ~30 kHz của driver. Điện trở nối tiếp tương đương (ESR) thấp của tụ gốm quan trọng hơn dung lượng của nó ở cấp độ này.
- **Tụ hóa 22 µF** xử lý dòng tiêu thụ dung lượng lớn giữa các chu kỳ chuyển mạch. Tụ 22 µF là đủ cho tải nhỏ của thiết bị này; nếu bạn nâng cấp động cơ lớn hơn hoặc thấy hiện tượng sụt áp khi đảo chiều nhanh, hãy nâng cấp tụ lên 470 µF trước khi đổi nguồn lớn hơn.

## Dây nối — 26 AWG lõi đơn

- Sử dụng một cỡ dây duy nhất cho cả tín hiệu và nguồn, bởi vì:
  - Dòng điện đỉnh của thiết bị là ~1 A; dây 26 AWG chịu được dòng 2.2 A liên tục trong đi dây khung vỏ (chassis wiring).
  - Sử dụng một loại dây dễ quản lý hơn việc chuẩn bị riêng các cỡ dây cho tín hiệu và nguồn, và sự khác biệt không ảnh hưởng ở mức dòng điện này.
  - Dây lõi đơn (solid-core) hàn chắc chắn và dễ dàng định vị trên bo mạch lỗ hơn dây nhiều sợi (stranded).
- Một cỡ dây mỏng hơn chuyên dụng cho I²C là không cần thiết với chiều dài cáp cảm biến AS5600 chỉ khoảng ~100 mm.

## Công tắc nguồn + jack cắm nguồn tròn

- Một **công tắc bập bênh SPST mắc nối tiếp** trên đường nguồn 12 V tiện lợi hơn nhiều so với việc rút jack nguồn. Việc bật tắt nguồn (power-cycling) là thao tác chẩn đoán thường gặp trong quá trình phát triển firmware.
- **Sự không khớp kích thước jack/phích cắm** (jack cái 5.5 × 2.1 mm so với phích cắm nguồn adapter 5.5 × 2.5 mm): sự khác biệt đường kính chân 0.4 mm tạo ra cảm giác cắm hơi lỏng nhưng tiếp xúc vẫn đáng tin cậy trong thực tế. Nếu bạn tìm được jack cắm cái 2.5 mm khớp với cùng mức giá, hãy ưu tiên chọn nó — nếu không thì sự không khớp này cũng vô hại.

## Những thứ cố ý *không* đưa vào BOM

- **Pin / mạch nâng áp (boost converter).** Thiết bị này luôn được cắm cáp kết nối máy tính để chạy pipeline RL; tính di động không phải là mục tiêu. Firmware PID độc lập *có thể* chạy bằng pin, nhưng việc thêm pin LiPo + mạch hạ áp buck sẽ tăng thêm diện tích cần chẩn đoán lỗi không cần thiết.
- **Diode TVS / diode bảo vệ trên đường nguồn.** Các bộ adapter cắm tường được sử dụng ở đây hoạt động rất ổn định; một mạch dập nhiễu RC (RC snubber) trên các dây dẫn động cơ hoặc một TVS tại VMOT sẽ là giải pháp bảo vệ tối đa nhưng không quá quan trọng đối với hoạt động ổn định thông thường.
- **Mạch chuyển đổi mức logic (Logic-level shifters).** Board Nano (5 V) + AS5600 (chấp nhận 3.3 V trên I²C với các điện trở pull-up nội bộ lên 5 V hoạt động tốt trên module này) — không cần mạch chuyển đổi mức logic. Các board AS5600 khác có thể khác; hãy kiểm tra điện áp pull-up của bo mạch breakout trước khi giả định.

## Liên kết liên quan

- [`BOM.md`](BOM.md) — tài liệu tham khảo mua sắm (nhà cung cấp, giá cả, số lượng).
- [`3d_printing.md`](3d_printing.md) — cấu hình in và kỹ thuật tạm dừng in để chèn đồng xu vào liên kết con lắc.
- [`sysid_runbook.md`](sysid_runbook.md) — quy trình đo lường xác thực chuỗi linh kiện điện tử hoạt động trơn tru từ đầu đến cuối.
