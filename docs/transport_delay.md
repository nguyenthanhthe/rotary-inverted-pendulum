# Độ trễ truyền thông (Transport delay): Cách giảm thiểu và cách đo lường

Thiết bị này từng có độ trễ truyền thông từ máy tính đến động cơ khoảng ~50 ms. Hiện tại con số này đã giảm xuống còn ~14 ms. Kết quả này đạt được nhờ ba lần sửa đổi, mỗi lần tập trung vào một lớp khác nhau của pipeline. Không có lần sửa đổi nào trong số ba lần này nhắm trực tiếp vào "độ trễ" *cơ bản*; chúng giải quyết các vấn đề khác và việc giảm độ trễ là một sản phẩm phụ (by-product) đi kèm.

## Pipeline (Luồng xử lý)

```
policy(obs) -> action  ┐
                       │  (1) Cổng serial USB     ~1–5 ms
                       ▼
                   Arduino loop  -- (2) Điều phối lệnh (Cmd dispatch)
                                 -- (3) Driver động cơ bước / ISR
                                 -- (4) Đáp ứng cơ học của động cơ
                       │
                       ▼
                  Cảm biến AS5600 -- (5) Đọc qua I²C       ~5 ms
                       │
                       ▼
                 policy obs(t+1)
```

Tổng độ trễ truyền thông khứ hồi (round-trip) = (1) + (2) + (3) + (4) + (5).

## Ba lần sửa đổi (theo trình tự thời gian)

| Ngày | Sửa đổi | Lớp tác động | Tại sao | Tác động phụ lên độ trễ |
|------------|--------------------------------------------------------------------------|------------------|--------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 03/05/2026 | `AccelStepper` → `FastAccelStepper` (commit `c46480d`) | (3) động cơ bước | Việc thăm dò (polling) thời gian xung bước trong AccelStepper bị dao động khi vượt quá 50 ksteps/s², gây mất bước; vị trí của firmware bị lệch so với thực tế. FastAccelStepper điều khiển chân STEP từ ngắt Timer1 OC1A ISR — tạo ra các xung chuẩn xác. Yêu cầu chuyển chân STEP_PIN sang pin 9. | Nhẹ: Thời gian ISR ổn định hơn so với việc thăm dò tuần tự, giúp giảm dao động thời gian ở lớp (3). Độ trễ trung vị hầu như không đổi. |
| 16/05/2026 | Ngữ nghĩa hành động: vị trí mục tiêu → gia tốc góc (commit `396c5d4`) | (3) động cơ bước | Các lệnh vị trí bắt buộc động cơ phải chạy lại đoạn tăng tốc hình thang (trapezoidal) để dừng tại mục tiêu ở mỗi tick điều khiển (tạo ra trễ cơ học 10–30 ms mỗi tick). Mô phỏng và thực tế bị lệch pha khi có cộng hưởng; điểm số upright mô phỏng đạt 0.14 so với thực tế đạt 0.73 trên các chuỗi hành động giống hệt nhau. Việc chuyển sang hàm `moveByAcceleration(int32 steps_s2, allow_reverse=true)` cho phép động cơ tích phân gia tốc liên tục qua các tick, với khả năng đảo chiều mượt mà khi đi qua điểm 0. | **Giảm mạnh.** Loại bỏ đoạn tăng tốc mục tiêu ở mỗi tick (10–30 ms). Đây là yếu tố đóng góp chính giúp giảm độ trễ từ 50 ms xuống còn ~14 ms. |
| 16/05/2026 | Quan sát bổ sung thêm `prev_action` (commit `cae2a1b`) | (1) chính sách | Ngay cả sau các lần sửa đổi 1 + 2, độ trễ ~14 ms còn lại vẫn khiến hệ thống trở thành POMDP dưới góc nhìn của chính sách — nó không thể biết liệu obs(t) phản ánh hành động action(t) hay action(t-1). Việc thêm `prev_action` vào quan sát giúp khôi phục tính chất Markov cho pipeline hành động. | Không ảnh hưởng trực tiếp. Không làm thay đổi độ trễ vật lý; chỉ giúp chính sách hiểu và xử lý được nó. |

## Phép đo hiện tại (16/05/2026)

Hai phương pháp đo, cùng một kết luận.

### Phương pháp 1 — Thử nghiệm bước sysid_accel (giữ con lắc cố định)

Được ghi lại bởi kịch bản `sysid_accel.py step` ở tần số ghi nhật ký 200 Hz trong khi điều khiển trực tiếp firmware qua các cuộc gọi `set_acceleration`. Động cơ đáp ứng trong vòng **một mẫu 200 Hz (≤ 5 ms)** khi lệnh gia tốc thay đổi dạng bước. Phương pháp này đo đạc các lớp (1) + (2) + (3) + (4) mà không tính đến lượt đọc I²C/Python.

### Phương pháp 2 — Khớp mô hình trễ nửa bước (half-step) từ nhật ký triển khai thực tế

Từ kịch bản `run_policy.py --log /tmp/pdfix.npz` chạy ở tần số điều khiển 35 Hz của chính sách. Chọn một bước mà gia tốc lệnh thay đổi đột ngột và quan sát lượng thay đổi vận tốc Δv sau đó hai bước:

```
idx 91: cmd trước = -37.5,  cmd = -149,  Δv quan sát được = -2.69 rad/s
        kỳ vọng nếu trễ 0 bước           -4.25
        kỳ vọng nếu trễ 1 bước           -1.07
        kỳ vọng nếu trễ ½ bước    ✓      -2.66
```

Kết quả khớp với **mô hình trễ ½ bước điều khiển (½-control-step delay model)** trong phạm vi nhiễu của bộ lọc. Ở tần số điều khiển 35 Hz, con số đó **tương đương ≈ 14 ms** độ trễ truyền thông hiệu dụng từ đầu đến cuối — bao gồm cả thời gian đọc cảm biến và thời gian tính toán của Python mà phương pháp 1 bỏ qua.

## Hệ quả

- **Phạm vi DR trong `pendulum_env.py` vẫn được hiệu chuẩn theo chế độ vị trí (position-mode).** Hằng số `DR_ACTION_DELAY_STEPS_RANGE = (1, 3)` được thiết lập khi độ trễ thực tế là ~50 ms (1–3 bước ở tần số 35 Hz). Thực tế sau khi áp dụng chế độ gia tốc chỉ còn khoảng ~½ bước.
- Lượng tử hóa trễ số nguyên (lấy mẫu 0 hoặc 1) là một phép khớp thô cho một độ trễ thập phân. Một DR **trễ hành động (action-lag)** liên tục (bộ lọc bậc một với hằng số tau ngẫu nhiên nằm trong khoảng ~[5, 20] ms) sẽ khớp trực tiếp hơn với động lực học thực tế của các lớp (3)+(4) và cung cấp cho bộ tối ưu hóa một gradient mượt mà hơn so với lấy mẫu rời rạc 0-hoặc-1.
- Các phạm vi trễ của Giai đoạn 2/3 trong chương trình học nên được thu hẹp để bao quanh giá trị thực tế ~14 ms, thay vì giá trị lịch sử 30–50 ms.
