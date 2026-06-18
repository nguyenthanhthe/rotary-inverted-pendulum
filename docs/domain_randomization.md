# Xáo trộn miền (Domain Randomization - huấn luyện mô phỏng)

Việc chuyển giao từ mô phỏng sang thực tế (sim-to-real) đối với thiết bị này là một bài toán ổn định vòng kín trên một động cơ bước điều khiển bằng vị trí. Bất cứ điều gì chúng ta bỏ sót trong mô hình mô phỏng (sim model), chính sách (policy) sẽ âm thầm overfit vào đó. Xáo trộn miền (Domain Randomization - DR) là cơ chế chúng tôi sử dụng để thu hẹp khoảng cách đó: chúng tôi huấn luyện hệ thống đối với một *phân phối (distribution)* các thiết bị hợp lý thay vì một thiết bị danh định (nominal) duy nhất, do đó chính sách có khả năng chống chịu (robust) với sự không khớp còn lại (residual mismatch) giữa các tham số nhận dạng được và trạng thái thực tế của phần cứng vào một ngày cụ thể.

Tài liệu này tóm tắt **những gì được xáo trộn, xáo trộn bao nhiêu và tại sao**, cùng với cách DR khớp vào lộ trình chương trình học (curriculum schedule) và vị trí của các nút cấu hình trong mã nguồn. Để có cái nhìn rộng hơn về kế hoạch RL (giai đoạn, quyết định, trạng thái), xem [`../RL_PLAN.md`](../RL_PLAN.md). Đối với đặc tả cấp độ chuyển trạng thái, xem [`rl_transitions.md`](rl_transitions.md). Đối với các phép đo nhận dạng hệ thống (sysid) thiết lập các giá trị giới hạn, xem [`sysid_runbook.md`](sysid_runbook.md).

## Cách kích hoạt

```bash
python train_sac.py --domain-randomization ...
```

Hoặc chạy toàn bộ chương trình học (curriculum):

```bash
./curriculum_train.sh <run-name-prefix>
```

Cờ lệnh này sẽ thay đổi một giá trị boolean duy nhất trên môi trường env ([`pendulum_env.py:248`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L248)). Khi tắt, môi trường env sẽ chạy một cách xác định (deterministic) dựa trên các tham số danh định của sysid — hữu ích cho việc debug nhưng **không bao giờ** dùng cho chính sách sẽ được triển khai thực tế.

Môi trường **eval env luôn ở trạng thái xác định** (tắt DR), do đó việc lựa chọn mô hình tốt nhất (best-model) trong quá trình huấn luyện sẽ theo dõi hiệu suất trên kịch bản tham chiếu vật lý danh định thay vì bị nhiễu do xáo trộn ngẫu nhiên giữa các mẫu. Xem [`train_sac.py:86`](../RotaryInvertedPendulum-python/src/rl/train_sac.py#L86).

## Những gì được xáo trộn

Tất cả các phạm vi được định nghĩa dưới dạng hằng số module trong [`pendulum_env.py:61-97`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L61-L97). Hầu hết được lấy mẫu **mỗi tập một lần** trong hàm [`_sample_dr_params`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L384); riêng dao động dt (dt-jitter) và nhiễu quan sát được lấy mẫu **mỗi bước một lần (per step)**.

### Tham số vật lý (mỗi tập một lần)

| Tham số | Phạm vi | Hằng số nguồn | Tại sao chọn khoảng này |
| ------------------------------------------- | -------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Khối lượng con lắc (Pendulum mass) | danh định × (1 ± 0.10) | `DR_PENDULUM_MASS_FRAC = 0.10` | Được thu hẹp từ ±0.20 sau khi kiểm tra chéo CAD (chi tiết Onshape với vật liệu được hiệu chỉnh mật độ PLA) khớp với giá trị sysid `m·d` trong phạm vi 1%; khối lượng là một phép đo trực tiếp duy nhất và phần không chắc chắn còn lại (nam châm chưa mô hình hóa, sự thay đổi độ đặc infill) nằm gọn trong khoảng ±10%. |
| Khoảng cách tâm khối (Pendulum COM distance) | danh định × (1 ± 0.10) | `DR_PENDULUM_COM_FRAC = 0.10` | Vị trí lắp đặt vòng bi và khối lượng đầu mút có thể thay đổi; ±10% bao quát được các lần lắp ráp lại khác nhau. |
| Ma sát khớp con lắc (nhớt + Coulomb) | danh định × [0.5, 2.0] | `DR_PENDULUM_FRICTION_MULT_RANGE = (0.5, 2.0)` | Ma sát phụ thuộc vào trạng thái dầu mỡ bôi trơn, nhiệt độ và độ chặt của vòng bi; cùng một hệ số nhân được áp dụng cho cả hai thành phần vì chúng có chung nguồn gốc là vòng bi. |
| Ma sát tĩnh của khớp động cơ (`frictionloss`) | [0.0, 0.005] N·m | `DR_MOTOR_FRICTIONLOSS_RANGE_N_M = (0.0, 0.005)` | Động cơ bước có mô-men giữ (detent torque) mà mô hình vị trí không nắm bắt được; giới hạn dưới bao gồm cả 0 để tương thích ngược với các chính sách Giai đoạn 2 được huấn luyện mà không có ma sát tĩnh. |

Các giá trị danh định đến từ [`sysid_params.json`](../RotaryInvertedPendulum-python/src/rl/sysid_params.json), được tạo ra bởi pipeline sysid Giai đoạn 0. **Quán tính của con lắc quanh tâm khối của chính nó** (`PENDULUM_I_COM_SWING_KG_M2` tại [`pendulum_env.py:71`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L71)) được hard-code từ CAD Onshape thay vì tính ngược lại từ sysid `I_axis − m·d²` — đây là một thuộc tính hình học của chi tiết chứ không phải một đại lượng thay đổi theo các lần lắp ráp lại, vì vậy nó không bị xáo trộn. MuJoCo tự động áp dụng định lý trục song song (parallel-axis theorem) từ `body_ipos`, cung cấp quán tính trục quay của mỗi tập là `m·d² + I_com_swing`. Trước đây điều này được ép buộc về ≈0 (xấp xỉ chất điểm); giá trị CAD (~8.06e-6 kg·m²) tăng thêm ~25% vào quán tính trục quay hiệu dụng ở mức danh định, khớp với giá trị đo được `I_axis`.

### Tính chân thực của cơ cấu chấp hành / vòng lặp điều khiển (mỗi tập một lần)

| Tham số | Phạm vi | Hằng số nguồn | Tại sao chọn khoảng này |
| ------------------------------------------ | -------------------------------- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Độ trễ bậc một của động cơ τ (Motor lag) | [0.0, 0.010] s | `DR_MOTOR_TAU_RANGE_S` | Giai đoạn 2.5 khớp với nhật ký phần cứng thực tế tìm thấy τ ≈ 0; chúng tôi vẫn quét tối đa 10 ms để duy trì khoảng dự phòng chống lại sự chậm trễ phụ thuộc vào tải trọng. |
| Trễ hành động (Action delay) | [4, 7] bước điều khiển | `DR_ACTION_DELAY_STEPS_RANGE` | Ở tần số 100 Hz, khoảng này bao quát độ trễ phần cứng ~50 ms đo được (5 bước) với biên sai số ±1 bước. Giai đoạn 2.6 đã mở rộng khoảng này sau khi giảm `MOTOR_ACCELERATION` (100k → 50k bước/s²) trên Arduino. Kịch bản chương trình học sẽ quy đổi giá trị này thành mili-giây vật lý ở tần số điều khiển thấp hơn — xem bên dưới. |
| Dao động dt của bước điều khiển (dt-jitter) | n_substeps × (1 ± 0.05) mỗi bước | `DR_CONTROL_DT_JITTER_FRAC = 0.05` | Về mặt thực nghiệm, đây là DR đơn lẻ quan trọng nhất. Nếu không có nó, SAC ở thời gian nghiêm ngặt sẽ rơi vào **điểm thu hút hiệu chỉnh tích cực (active-correction attractor)** (động cơ chuyển động ±0.5 rad ngay cả khi đã cân bằng); có nó, SAC tìm thấy **điểm thu hút tĩnh với hành động tối thiểu (calm minimal-action attractor)** thống trị hiệu suất thực tế. Xem [`control_rate_selection.md`](control_rate_selection.md) phần "điểm thu hút tĩnh so với tích cực". |

### Nhiễu quan sát (mỗi bước một lần)

| Tham số | Phạm vi | Hằng số nguồn | Tại sao chọn khoảng này |
| --------------------------- | ------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Lượng tử hóa góc con lắc | khớp với AS5600 LSB (2π / 4096 rad) | `PENDULUM_LSB_RAD` | Mô phỏng độ phân giải 12-bit của cảm biến mã hóa vòng quay. Luôn được áp dụng khi DR bật. |
| Nhiễu vị trí σ | 0.005 rad | `DR_OBS_NOISE_STD_POS_RAD` | Mô phỏng dao động hiệu sai phân và nhiễu encoder quan sát được trên thiết bị. Được cộng vào vị trí động cơ và góc con lắc. |
| Nhiễu vận tốc σ | 0.05 rad/s | `DR_OBS_NOISE_STD_VEL_RAD_S` | Vận tốc được tính vi phân sai số từ tín hiệu vị trí bị nhiễu; σ được chọn để khớp với dao động quan sát được tại tần số cắt của bộ lọc trên thiết bị này. |

## Các giai đoạn của chương trình học (Curriculum staging)

Huấn luyện qua ba giai đoạn ([`curriculum_train.sh`](../RotaryInvertedPendulum-python/src/rl/curriculum_train.sh)) mang lại độ tin cậy cao hơn so với huấn luyện một lần ở toàn bộ độ rộng DR. Mỗi giai đoạn sử dụng tùy chọn `--resume` từ giai đoạn trước để tích lũy khả năng.

Hai tham số được thay đổi dần (annealed) qua các giai đoạn là **trễ hành động (action delay)** và **dao động dt (dt jitter)**. Các phạm vi DR khác được giữ cố định cho tất cả các giai đoạn vì chính sách được hưởng lợi từ việc quan sát chúng ngay từ đầu.

| Giai đoạn | Trễ hành động (vật lý) | Động cơ τ | Dao động dt | Số bước (Steps) |
| -------------- | ----------------------- | ---------- | --------- | ----------------- |
| **1 — dễ** | [0, 20] ms | [0, 5] ms | ±20 % | `STEPS_PER_STAGE` |
| **2 — trung bình** | [20, 50] ms | [0, 10] ms | ±10 % | `STEPS_PER_STAGE` |
| **3 — cuối cùng** | [30, 60] ms | [0, 10] ms | ±5 % | `STEPS_PER_STAGE` |

Lưu ý:

- Tập lệnh chuyển đổi các khoảng mili-giây vật lý thành số lượng bước (step counts) nguyên ở tần số `CONTROL_FREQ` đã cấu hình. Ở tần số thấp (ví dụ: 35 Hz), giai đoạn 2 và 3 có thể trùng nhau sau khi làm tròn; trong trường hợp đó, giai đoạn 3 sẽ được bỏ qua và chính sách cuối cùng là mô hình tốt nhất của giai đoạn 2. Tập lệnh sẽ ghi lại thông tin này một cách rõ ràng.
- Việc giảm dần dao động dt (dt-jitter) là có chủ ý: dao động cao ở giai đoạn đầu buộc chính sách phải thoát khỏi điểm thu hút hiệu chỉnh tích cực; dao động thấp hơn ở giai đoạn sau giúp chính sách thích ứng chuyên sâu với thời gian thực tế triển khai.
- Giai đoạn 3 bao quanh độ trễ ~50 ms của phần cứng với khoảng dự phòng khoảng một bước ở mỗi bên ở tần số 35 Hz (tần số chuẩn cho thiết bị này).

## Đa dạng hóa khởi động lại (Reset diversity - không hẳn là DR nhưng liên quan)

Bản thân hàm `reset()` thêm sự đa dạng về điều kiện ban đầu, điều này rất cần thiết để chính sách học cách phục hồi từ các điểm bắt đầu bất kỳ:

- **Bắt đầu động cơ**: phân phối đều trong khoảng ±0.7 × giới hạn an toàn của động cơ (≈ ±88°). Giúp giữ trạng thái reset không chạm vào giới hạn cắt ±125° trong khi bao phủ hầu hết phạm vi hoạt động. Nếu không có điều này, chính sách sẽ không bao giờ thực hành quay trở lại từ giới hạn và sẽ bị kẹt ở đó khi triển khai ([`pendulum_env.py:362-374`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L362-L374)).
- **Bắt đầu con lắc**: treo tự do ± 0.05 rad (nhiễu nhỏ xung quanh góc nghỉ tự nhiên).

Điều này luôn được bật bất kể tùy chọn `--domain-randomization` có được chọn hay không vì nó phục vụ việc bao phủ không gian trạng thái huấn luyện, chứ không phải khả năng chống chịu sai số mô hình.

## Vị trí của các nút cấu hình trong mã nguồn

| Nút tinh chỉnh | Mặc định | Nơi thiết lập / đọc |
| ----------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hằng số module (`DR_*`) | Như trên | [`pendulum_env.py:61-97`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L61-L97) |
| Ghi đè cho từng instance (chương trình học) | None → sử dụng hằng số module mặc định | `RotaryInvertedPendulumEnv.__init__` ([`pendulum_env.py:255-284`](../RotaryInvertedPendulum-python/src/rl/pendulum_env.py#L255-L284)) |
| Ghi đè qua giao diện dòng lệnh (CLI overrides) | Không đặt → sử dụng hằng số module mặc định | Các cờ lệnh của `train_sac.py` `--dr-tau-min/max`, `--dr-delay-min/max`, `--dr-dt-jitter-frac` ([`train_sac.py:198-226`](../RotaryInvertedPendulum-python/src/rl/train_sac.py#L198-L226)) |
| Giá trị của các giai đoạn chương trình học | Các mục tiêu ms được hard-code | [`curriculum_train.sh`](../RotaryInvertedPendulum-python/src/rl/curriculum_train.sh) |

## Cách chỉnh sửa các phạm vi xáo trộn

Khi phần cứng thực tế cho thấy điều gì đó mà mô phỏng đã bỏ sót:

1. **Hãy chạy lại sysid trước.** Nếu các tham số danh định bị trôi lệch (vòng bi mới, thay động cơ, v.v.), hãy cập nhật các giá trị đó trước khi mở rộng DR — DR không thay thế cho việc nhận dạng hệ thống tốt, nó chỉ bao quanh phần sai số còn lại của nó. Xem [`sysid_runbook.md`](sysid_runbook.md).
2. **Mở rộng phạm vi DR tương ứng** trong tệp `pendulum_env.py`. Hãy giữ các phạm vi ở mức an toàn — chúng nên bao quanh thực tế đo được với một khoảng dự phòng, chứ không nên bao gồm các chế độ vật lý phi lý (chúng chỉ làm chậm quá trình huấn luyện và không dạy cho chính sách điều gì hữu ích).
3. **Huấn luyện lại từ đầu qua toàn bộ chương trình học.** Việc tiếp tục huấn luyện (resume) từ các checkpoint của giai đoạn 3 là không an toàn khi phân phối nền tảng thay đổi; giai đoạn 1 sẽ thích ứng với các điều cơ bản nhanh nhất.
4. **Xác thực bằng `eval_randomized.py`** để kiểm tra nhanh khả năng chống chịu của chính sách trên phạm vi mới trước khi triển khai thực tế.

## Liên kết liên quan

- [`control_rate_selection.md`](control_rate_selection.md) — tại sao dao động dt lại là nút DR chịu tải quan trọng nhất trên thiết bị này.
- [`rl_transitions.md`](rl_transitions.md) — đặc tả chuyển trạng thái `(s, a, r, s')` mà DR xáo trộn.
- [`sysid_runbook.md`](sysid_runbook.md) — quy trình đo lường các giá trị danh định mà DR bao quanh.
- [`async_control_architecture.md`](async_control_architecture.md) — cách runtime triển khai bảo toàn các giả định thời gian mà DR được huấn luyện.
