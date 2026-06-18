# Đặc tả chuyển trạng thái RL (RL Transitions)

Tài liệu này giải thích chi tiết những gì một bộ chuyển trạng thái `(s, a, r, s')` trong dự án này thực sự chứa từ đầu đến cuối. Tài liệu này đóng vai trò tham chiếu cho cả môi trường mô phỏng (`pendulum_env.py`) và thực tế (`real_env.py`) — chúng được thiết kế giống hệt nhau để một checkpoint được huấn luyện trong mô phỏng có thể trực tiếp tiếp tục học trên thiết bị thực mà không cần bất kỳ bộ chuyển đổi nào.

## Tóm tắt thiết lập trong một đoạn văn

Hệ con lắc ngược quay (kiểu Furuta). Một cánh tay ngang quay trong mặt phẳng nằm ngang, được dẫn động bởi một động cơ bước (góc "motor"). Ở đầu cánh tay đó treo một con lắc quay tự do (góc "pendulum"). Mục tiêu là swing-up (đánh đu) con lắc từ vị trí treo thẳng đứng xuống dưới lên vị trí thẳng đứng hướng lên trên và giữ nó cân bằng ở đó. Mỗi 10 ms, chính sách quan sát trạng thái hiện tại, đưa ra một hành động điều khiển nhỏ, và hệ thống chuyển dịch sang trạng thái tiếp theo.

## Các quy ước và hệ tọa độ

- **Góc con lắc θ (Pendulum angle θ)**: 0 nghĩa là **thẳng đứng hướng lên** (upright), ±π nghĩa là treo thẳng đứng xuống dưới. Chúng tôi sử dụng θ thay vì góc khớp thô φ của MuJoCo ở mọi nơi trong quan sát (observation) và phần thưởng (reward), bởi vì nó giúp biểu diễn mục tiêu một cách đơn giản là `θ=0`. Về mặt nội bộ: `θ = wrap_pi(φ − π)`.
- **Góc động cơ (Motor angle)**: 0 là tâm được hiệu chuẩn lúc khởi động. Giá trị dương = quay ngược chiều kim đồng hồ khi nhìn từ trên xuống. Giới hạn cơ khí dừng vật lý ở mức ±135°; chúng tôi cắt các mục tiêu lệnh trong khoảng ±125° để chính sách không bao giờ yêu cầu đâm vào giới hạn dừng.
- **Tần số điều khiển**: có thể cấu hình (`control_freq_hz`). Động lực học mô phỏng tích phân ở tần số nội bộ 1 kHz; môi trường thực tế sẽ điều phối tốc độ thời gian thực để khớp với tần số này. Việc lựa chọn tần số phụ thuộc vào từng thiết bị và bị giới hạn bởi băng thông động cơ cũng như động lực học của con lắc — xem [`control_rate_selection.md`](control_rate_selection.md). Việc áp đặt tần số này trong thời gian chạy (runtime) được mô tả trong [`async_control_architecture.md`](async_control_architecture.md).

## Trạng thái quan sát `s` — Những gì chính sách nhìn thấy (5 số thực float)

```
s = [motor_pos, sin(θ), cos(θ), motor_vel, pendulum_vel]
```

| Thành phần | Đơn vị | Phạm vi | Ý nghĩa |
|---|---|---|---|
| `motor_pos` | rad | ±2.36 (= ±135°) | Góc cánh tay hiện tại so với tâm |
| `sin(θ)` | — | [−1, 1] | Sine của góc lệch con lắc so với phương thẳng đứng |
| `cos(θ)` | — | [−1, 1] | Cosine của góc lệch con lắc so với phương thẳng đứng. **= 1 tại mục tiêu cân bằng**, = −1 khi treo thẳng đứng xuống dưới |
| `motor_vel` | rad/s | ±200 | Vận tốc góc của cánh tay |
| `pendulum_vel` | rad/s | ±200 | Vận tốc góc của con lắc |

Tại sao sử dụng sine + cosine thay vì góc θ trực tiếp? Góc θ bị nhảy giá trị tuần hoàn tại ±π, vì vậy mạng chính sách sẽ thấy một điểm gián đoạn ngay cạnh một trong những điểm hoạt động của nó (khi treo thẳng đứng xuống dưới). Bộ `(sin θ, cos θ)` là một biểu diễn liên tục trên đường tròn đơn vị — không bị nhảy giá trị, không có vách đá gradient.

**Cách trạng thái `s` được xây dựng**:

- **Trong mô phỏng** (`pendulum_env.py::_obs`): đọc trực tiếp `qpos`/`qvel` từ MuJoCo. Khi bật xáo trộn miền (DR), chúng tôi bổ sung việc lượng tử hóa góc con lắc theo LSB của cảm biến AS5600 (12-bit, ~0.0015 rad) và chèn nhiễu Gauss nhỏ vào vị trí và vận tốc — để quy trình quan sát trong mô phỏng khớp với những gì thiết bị thực tế cung cấp.
- **Trên thiết bị thực** (`real_env.py::_build_obs`): truy vấn LowLevelServer qua cổng serial → đảo ngược quy ước dấu của firmware → tính hiệu sai phân (finite-difference) để ước lượng vận tốc và đưa chúng qua một bộ lọc thông thấp 20 Hz. Bộ lọc này cực kỳ quan trọng: tính hiệu sai phân thô ở tần số 100 Hz trên một encoder bị nhiễu là không thể sử dụng được.

## Hành động `a` — Đầu ra của chính sách (1 số thực float)

```
a ∈ [−1, 1]
```

Hành động này **không phải là mô-men xoắn hay vị trí tuyệt đối**. Nó là một lượng thay đổi *nudge (delta)* đã chuẩn hóa được áp dụng cho vị trí động cơ mục tiêu ở mỗi bước:

```
motor_target ← clip(motor_target + a · max_action_delta_rad, ±125°)
```

Do đó, chính sách lái động cơ bằng cách đưa ra các nhích nhỏ (nudges) ở mỗi bước. Hàm `clip` áp đặt giới hạn mềm cho động cơ để chính sách không thể ra lệnh đâm vào giới hạn dừng vật lý, ngay cả khi nó muốn.

Tham số `max_action_delta_rad` là một trong hai nút tinh chỉnh đi đôi với nhau (nút kia là `control_freq_hz`). Tích của chúng là *slew rate* (tốc độ thay đổi mục tiêu) tính bằng rad/s, thông số này phải tuân thủ băng thông của động cơ — xem [`control_rate_selection.md`](control_rate_selection.md) để biết cơ sở lý thuyết và quy trình thực hiện.

Cách biểu diễn hành động này mang lại hai lợi ích chính:

1. Firmware động cơ bước (AccelStepper) chấp nhận các mục tiêu vị trí, không chấp nhận mô-men xoắn. Việc ánh xạ hành động trực tiếp thành delta vị trí khớp với giao tiếp phần cứng.
2. Đảm bảo tính mượt mà về mặt cấu trúc: giữa hai bước liên tiếp, vị trí lệnh chỉ có thể thay đổi tối đa 0.1 rad, bất kể chính sách xuất ra giá trị cực đoan thế nào. Điều này giới hạn tốc độ thay đổi (slew rate) của cơ cấu chấp hành trong trường hợp xấu nhất và bảo vệ động cơ trong quá trình khám phá (exploration) khi huấn luyện.

## Động lực học chuyển trạng thái — đi từ `s` sang `s'`

Trong **mô phỏng** (mỗi bước):

1. Áp dụng hàng đợi trễ hành động (DR lấy mẫu một khoảng trễ 0–N bước được nhân tỉ lệ để bao quanh độ trễ truyền thông đo được của thiết bị; hành động có hiệu lực bây giờ có thể là hành động chính sách đã chọn từ vài bước trước — mô phỏng RTT của serial + đoạn tăng tốc của AccelStepper).
2. Cập nhật vị trí lệnh `motor_target` với hành động (đã bị trễ).
3. Áp dụng độ trễ bậc một của động cơ (DR lấy mẫu τ ∈ [0, 10] ms): giá trị `motor_applied` cấp cho MuJoCo sẽ bám theo mục tiêu lệnh với một hàm mũ có hằng số thời gian τ. Với τ=0 thì phản hồi là tức thời.
4. Chạy một bước vật lý của MuJoCo cho một chu kỳ điều khiển với các bước phụ (substeps) 1 ms.
5. Đọc ra trạng thái mới, xây dựng `s'`, tính toán phần thưởng (reward).
6. Tập huấn luyện kết thúc (terminate) nếu động cơ đâm vào giới hạn dừng vật lý ở ±135° (chịu phạt một lượng `−5`); cắt ngắn tập (truncate) sau khoảng thời gian `episode_length_s` (mặc định là 8 giây).

Trên **thiết bị thực** (mỗi bước):

1. Cập nhật vị trí lệnh `motor_target` với hành động (không có hàng đợi trễ hành động — phần cứng thực tế chính là độ trễ).
2. Gửi lệnh `set_target(motor_target)` qua cổng serial.
3. Chờ cho đến tick tiếp theo (điều phối thời gian thực khớp với tần số điều khiển).
4. Đọc trạng thái từ thiết bị: `(time_us, motor_pos, pendulum_pos)`.
5. Tính hiệu sai phân + lọc thông thấp để ước lượng vận tốc.
6. Xây dựng `s'`, tính toán phần thưởng.
7. Kết thúc tập khi |motor_pos| ≥ 135° (firmware cũng áp đặt giới hạn này ở phần cứng làm chốt chặn cuối cùng); cắt ngắn tập sau khoảng thời gian `episode_length_s` (mặc định là 6 giây trong quá trình tinh chỉnh).

Cầu nối sim-to-real quan trọng ở đây là **độ trễ truyền thông của firmware, đoạn tăng tốc và ma sát của động cơ bước chính là những gì các tham số `action_delay_steps`, `motor_tau_s` và ma sát khớp của mô phỏng đang mô hình hóa**. Xáo trộn miền (DR) lấy mẫu các giá trị này trên các phạm vi hợp lý ở mỗi tập để chính sách được tiếp xúc với đủ sự biến đổi để có thể xử lý thiết bị thực tế.

## Phần thưởng `r` (Reward r) — những gì chúng ta đang khuyến khích

Phần thưởng hiện tại tuân theo **dạng chi phí toàn phương Quanser tiêu chuẩn (Quanser quadratic-cost form)** (phổ biến trong các tài liệu về con lắc Furuta):

```
r = −[ θ² + k_θ̇·θ̇² + k_α·α² + k_α̇·α̇² + k_a·a² ]
```

trong đó:

- **θ** = góc lệch con lắc so với phương thẳng đứng (rad). Thành phần chi phối chính: `θ²` bằng 0 tại mục tiêu và ≈ π² ≈ 9.87 khi treo thẳng đứng xuống dưới.
- **θ̇** = vận tốc con lắc (pendulum_vel). Trọng số nhỏ `k_θ̇=0.001` để ngăn cản việc con lắc quay tròn liên tục qua điểm thẳng đứng.
- **α** = góc động cơ (rad, biến này bị đặt tên nhầm từ Quanser trong mô phỏng). Trọng số `k_α=0.5` giữ chính sách hoạt động gần tâm.
- **α̇** = vận tốc động cơ (motor_vel). Trọng số `k_α̇=0.005` ngăn cản cánh tay di chuyển quá điên cuồng.
- **a** = hành động (action) ∈ [−1, 1]. Trọng số `k_a=0.05` phạt nhẹ đối với các thao tác điều khiển giật cục.

Phần thưởng **luôn có giá trị không dương** — đạt cực đại bằng 0 khi con lắc đứng yên hoàn toàn ở tâm thẳng đứng và không có hoạt động của động cơ, và khoảng −10 mỗi bước khi treo thẳng đứng xuống dưới. Thuật toán SAC xử lý các phần thưởng âm rất tốt, và tín hiệu hoàn toàn âm giúp định hướng gradient "bớt âm hơn" về phía thẳng đứng một cách rõ ràng.

**Những gì chính sách thực tế học được để làm:**

- *Khi ở xa vị trí thẳng đứng*: Đánh đu cánh tay qua lại. Thành phần `θ²` khuyến khích việc đưa con lắc lên cao; các hình phạt `α²` và `α̇²` giữ cho chuyển động đánh đu nằm trong giới hạn để không đập vào các biên an toàn.
- *Khi ở gần vị trí thẳng đứng*: Giữ yên. Thành phần chi phối `θ²` tiến gần về không ở đó, chỉ còn lại các hình phạt vận tốc/hành động nhỏ làm sai số dư — do đó chính sách được khuyến khích duy trì bất kỳ trạng thái nào gần với (θ=0, θ̇=0, α=0, α̇=0).

Nếu chính sách đưa được con lắc về gần thẳng đứng nhưng không giữ yên được, chi phí sẽ bị chi phối bởi `k_θ̇·θ̇²`. Nếu nó cân bằng được nhưng cánh tay bị trôi lệch xa tâm, chi phí sẽ bị chi phối bởi `k_α·α²`. Các trọng số này là những gì định hình chính sách đi từ việc "bắt giữ loạng choạng" hướng tới "giữ thăng bằng mượt mà".

## Ranh giới tập huấn luyện (Episode boundaries)

| Sự kiện | Hành động | Khi nào |
|---|---|---|
| **Khởi động lại (Reset)** | Mô phỏng đặt con lắc treo thẳng xuống dưới với một nhiễu nhỏ, động cơ ở vị trí ngẫu nhiên `±0.7·motor_safe_limit`. Thiết bị thực ngắt động cơ, chờ `reset_settle_s` để con lắc dừng hẳn một cách tự do, kích hoạt lại động cơ ở vị trí hiện tại. | Mỗi tập |
| **Kết thúc (Termination)** | `terminated = True`, phần thưởng nhận thêm một hình phạt cuối cùng là `−5`. Đóng tập. | Đâm vào giới hạn dừng vật lý (`|motor_pos|≥135°`) |
| **Cắt ngắn (Truncation)** | `truncated = True`, phần thưởng không bị ảnh hưởng. Đóng tập. | Chạm giới hạn thời gian (8 giây trong mô phỏng, mặc định 6 giây trên thiết bị thực) |

Cắt ngắn tập chỉ ghi nhận giới hạn thời gian — giá trị tương lai vẫn được ước lượng bình thường. Kết thúc tập báo hiệu rằng giá trị tương lai sẽ giảm về 0 ("game over") và được dành riêng cho kết quả xấu là đập vào giới hạn dừng vật lý.

## Tại sao cách biểu diễn này hoạt động hiệu quả

Một vài lựa chọn thiết kế không hiển nhiên được làm rõ:

- **Sử dụng `(sin θ, cos θ)` thay vì θ**: tránh việc nhảy giá trị tuần hoàn khi xoay vòng. Chính sách nhìn thấy một không gian trạng thái mịn liên tục, không phải một hàm bước nhảy.
- **Hành động là delta vị trí mục tiêu, không phải vị trí mục tiêu tuyệt đối**: bằng cách tích phân đầu ra của chính sách, chúng tôi cho phép chính sách *lái* động cơ thay vì *nhảy cóc*. Tốc độ thay đổi (slew rate) được giới hạn; động cơ không bao giờ bị ra lệnh dịch chuyển tức thời.
- **Phần thưởng hoàn toàn âm**: tạo ra bề mặt tối ưu hóa đơn giản hơn so với việc trộn lẫn giá trị dương/âm. Phần thưởng entropy (entropy bonus) của SAC xử lý tốt việc khám phá; chúng tôi không cần thêm các phần thưởng định hình dương (positive shaping bonuses).
- **Cùng một môi trường cho cả mô phỏng và thực tế**: chi phí chuyển giao bằng không khi tải checkpoint. Replay buffer trong quá trình tinh chỉnh Giai đoạn 4 được lấp đầy bởi các bộ chuyển trạng thái thực tế `(s, a, r, s')` trông giống hệt về mặt cấu trúc so với các bộ trong mô phỏng mà chính sách đã học từ trước.

## Vị trí trong mã nguồn

| Tệp | Chức năng |
|---|---|
| `pendulum_env.py` | Môi trường mô phỏng đầy đủ — mô hình MJCF, DR, trễ hành động, độ trễ động cơ, phần thưởng. Tài liệu tham khảo chuẩn. |
| `real_env.py` | Phiên bản phần cứng. Gương soi thiết kế của `pendulum_env.py` khớp hoàn hảo về quan sát, hành động và phần thưởng. |
| `run_policy.py` | Client chỉ dùng để triển khai thực tế. Cùng pipeline quan sát như `real_env.py`, không có tính năng học máy. |
| `async_control.py`, `finetune_async.py` | Runtime *tạo ra* các chuyển trạng thái trong quá trình tinh chỉnh ở tần số nghiêm ngặt. Chi tiết bên trong nằm ngoài phạm vi tài liệu này — xem [`async_control_architecture.md`](async_control_architecture.md). |
| `finetune_real.py` | Lớp chuyển tiếp tương thích ngược → chuyển tiếp cuộc gọi tới `finetune_async.main`. |

Hãy đọc đồng thời hàm `pendulum_env.py::step` và `pendulum_env.py::_obs` để xem bước chuyển trạng thái chuẩn trong mô phỏng; đọc `real_env.py::step` và `real_env.py::_build_obs` để xem luồng tương tự đối với phần cứng thực tế.
