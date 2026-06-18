# Hướng dẫn chạy từ đầu đến cuối (End-to-End Runbook)

Cách đưa một thiết bị mới chế tạo từ trạng thái "chưa có chính sách (policy) nào" đến "tự cân bằng độc lập trên Nano, không cần kết nối máy tính". Mỗi bước liệt kê lệnh thực hiện, thời gian ước tính và điều kiện cần cho bước tiếp theo.

Để hiểu rõ *tại sao* pipeline được thiết kế theo cách này, xem [`../RL_PLAN.md`](../RL_PLAN.md). Để biết *cách hoạt động của một bước chuyển đổi (transition)*, xem [`rl_transitions.md`](rl_transitions.md).

```
    ┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────────┐
    │ 0 sysid  │───▶│ 1 sim    │───▶│ 2 fine-tune  │───▶│ 3 test       │
    │  recordings    curriculum     real-rig async      teacher tethered
    └──────────┘    └──────────┘    └──────────────┘    └──────┬───────┘
                                                               │ điểm số ≥ 0.9?
                                                               ▼
                                    ┌──────────────────────┐    ┌──────────┐
                                    │ 6 flash + verify     │◀───│ 4 distill│
                                    │ trên board độc lập   │    │ student  │
                                    └──────────────────────┘    └────┬─────┘
                                                                     │
                                                                ┌────▼──────┐
                                                                │ 5 test    │
                                                                │ student   │
                                                                │ tethered  │
                                                                └───────────┘
```

Nếu bạn chỉ cần hành vi dựng thẳng/cân bằng và chấp nhận kết nối máy tính trong khi chạy: hãy dừng lại sau bước 3. Các bước 4–6 chỉ tồn tại để loại bỏ dây kết nối máy tính (tether).

## Yêu cầu trước

- Máy tính phát triển chạy macOS / Linux có cài đặt `arduino-cli`, core `arduino:avr`, và các thư viện `AS5600` (RobTillaart) + `FastAccelStepper`.
- Môi trường Python được thiết lập theo [`../RotaryInvertedPendulum-python/README.md`](../RotaryInvertedPendulum-python/README.md):
  `mamba activate rotary-inverted-pendulum`.
- Thiết bị được đấu dây với **chân STEP nối với pin 9**, chân DIR nối với pin 2, chân ENABLE nối với pin 5, cảm biến AS5600 nối với I²C (A4/A5). Chân pin 9 là bắt buộc đối với thư viện FastAccelStepper trên vi điều khiển ATmega328 và cũng hoạt động tốt với thư viện AccelStepper — xem tệp [`../NEXT_STEPS.md`](../NEXT_STEPS.md).

Trong các lệnh dưới đây, hãy thay thế `/dev/cu.usbserial-1130` bằng cổng phù hợp mà lệnh `arduino-cli board list` hiển thị cho board Nano của bạn.

## 0. Nhận dạng hệ thống (System identification) — Đo đạc thiết bị

Xác định cố định các tham số động lực học. Đầu ra là tệp `sysid_params.json`, được đọc bởi `pendulum_env.py` để xây dựng môi trường mô phỏng.

Quy trình chi tiết: [`sysid_runbook.md`](sysid_runbook.md). Hãy chạy lại quy trình này bất cứ khi nào thiết bị có thay đổi về cơ khí (thay vòng bi mới, lắp lại cánh tay, thay đổi chế độ vi bước, thay động cơ).

## 1. Huấn luyện chính sách giáo viên (teacher) trong mô phỏng — Chương trình học (Curriculum)

Quy trình chuẩn của dự án huấn luyện một SAC actor qua 3 giai đoạn DR với mức độ thực tế tăng dần. Thời gian thực hiện mất khoảng ~25 phút trên MacBook (chạy bằng CPU, một env duy nhất). Đầu ra là tệp `runs/<run-name>/last.zip`.

```bash
cd RotaryInvertedPendulum-python/src/rl
bash curriculum_train.sh
```

Tập lệnh `curriculum_train.sh` đọc tệp `sysid_params.json`, tính toán các phạm vi DR từ các đơn vị thời gian vật lý, chạy qua 3 giai đoạn huấn luyện và ghi lại chính sách cuối cùng cùng với các điểm lưu (checkpoints). Tần số điều khiển `control_freq_hz` mặc định là **35 Hz** ở mọi thành phần — xem [`control_rate_selection.md`](control_rate_selection.md) để biết lý do.

## 2. Tinh chỉnh trên thiết bị thực — Bất đồng bộ (Async)

Chính sách huấn luyện trong mô phỏng sẽ chưa thể cân bằng ngay lập tức trên phần cứng thực tế (do khoảng cách giữa mô phỏng và thực tế - sim-to-real gap). Quá trình tinh chỉnh bất đồng bộ (async fine-tuning) sẽ thu hẹp khoảng cách đó sau khoảng ~30–80 tập huấn luyện thực tế (mất khoảng ~10–25 phút thời gian thực).

Đầu tiên, nạp chương trình LowLevelServer để máy tính có thể điều khiển thiết bị qua giao thức nhị phân:

```bash
arduino-cli compile --upload -p /dev/cu.usbserial-1130 \
    --fqbn arduino:avr:nano:cpu=atmega328 \
    RotaryInvertedPendulum-arduino/LowLevelServer
```

Sau đó, chạy bộ điều phối bất đồng bộ. Tùy chọn `--resume-buffer` không bắt buộc trong phiên chạy đầu tiên; ở các phiên chạy tiếp theo, hãy trỏ nó tới tệp `replay_buffer.pkl` của phiên chạy trước để tiếp tục tích lũy dữ liệu chuyển trạng thái thực tế:

```bash
cd RotaryInvertedPendulum-python/src/rl

# Phiên tinh chỉnh đầu tiên
python finetune_async.py \
    --policy runs/<sim-run>/last.zip \
    --port /dev/cu.usbserial-1130 \
    --episodes 50 \
    --run-name async_v1

# Các phiên tiếp theo, tiếp tục từ buffer cũ
python finetune_async.py \
    --policy runs/async_v1/last.zip \
    --resume-buffer runs/async_v1/replay_buffer.pkl \
    --port /dev/cu.usbserial-1130 \
    --episodes 30 \
    --run-name async_v1_extend
```

Chi tiết kiến trúc: [`async_control_architecture.md`](async_control_architecture.md).

Bộ điều phối sẽ tắt kích hoạt động cơ trong khoảng thời gian `--reset-settle-s` (mặc định là 5) giữa các tập để con lắc dừng lại một cách thụ động. **Hãy lắng nghe** tiếng động cơ trong vài tập đầu tiên — tiếng vo vo mượt mà là bình thường, nếu có tiếng rè rè/kẹt kẹt nghĩa là bị mất bước (hãy giảm `MOTOR_ACCELERATION` trong `LowLevelServer.ino` và `RLControl.ino` từ 50 k xuống 30 k và nạp lại chương trình).

## 3. Kiểm tra chính sách giáo viên (teacher) trên thiết bị — Có kết nối máy tính (Tethered)

Xác nhận xem giáo viên đã tinh chỉnh có thực sự cân bằng được hay không trước khi dành thêm thời gian cho nó. Bước này rất nhanh (chỉ mất 30 giây chạy thiết bị).

```bash
python run_policy.py \
    --policy runs/async_v1_extend/last.zip \
    --port /dev/cu.usbserial-1130 \
    --duration-s 30
```

Quan sát giá trị đại diện cho trạng thái thẳng đứng (`upright` proxy) được in ra ở cuối lượt chạy:
- **Trung bình (mean) ≥ 0.9** → giáo viên hoạt động tốt, tiến hành bước 4 để bỏ kết nối máy tính.
- **Trung bình 0.85–0.9** → chấp nhận được; có thể tiếp tục nhưng chạy thêm vài tập tinh chỉnh nữa (quay lại bước 2 với `--resume-buffer`) thường sẽ giúp tăng khả năng chống chịu nhiễu.
- **Trung bình < 0.85** → quá trình tinh chỉnh không hội tụ. Cần chẩn đoán lỗi trước khi chưng cất: kiểm tra các ý tưởng cải tiến chính sách [policy improvement ideas](policy_improvement_ideas.md), xem xét chạy lại sysid, hoặc cân nhắc tăng tham số `--gradient-steps`.

Nếu bạn chấp nhận giữ kết nối với máy tính, **bạn có thể dừng lại ở đây**. Chính sách giáo viên chạy ở tần số 35 Hz qua kết nối serial USB rất tốt.

## 4. Chưng cất (Distill) — Thu nhỏ actor cho Nano

Chính sách giáo viên SAC sau tinh chỉnh có kích thước 67 nghìn tham số (≈ 270 KB số thực float32) — quá lớn đối với bộ nhớ flash 32 KB của Nano. Chúng tôi tiến hành chưng cất thành một chính sách học sinh (student) có cấu trúc mạng 5→32→32→1 (kích thước ≈ 5 KB). Bước này mất khoảng ~1 phút:

```bash
python distill.py \
    --teacher runs/async_v1_extend/last.zip \
    --buffer  runs/async_v1_extend/replay_buffer.pkl \
    --out-dir runs/async_v1_extend/distill_h32_aug
```

Kịch bản `distill.py` sẽ:
1. Tải mô hình giáo viên + replay buffer (các quan sát thực tế từ thiết bị).
2. Đánh giá lại hành động xác định (deterministic-mean) của giáo viên trên mỗi quan sát.
3. Tăng cường dữ liệu bằng 100 nghìn lượt chạy của giáo viên trong môi trường mô phỏng DR (giúp ích khi buffer thực tế còn nhỏ/thưa thớt).
4. Huấn luyện một mạng MLP nhỏ thông qua hồi quy có giám sát (supervised regression) với đầu ra tanh.
5. Kiểm tra chéo độ khớp kết quả giữa numpy và PyTorch (để phát hiện lỗi xuất mô hình).

Tiêu chí nghiệm thu cho học sinh: sai số bình phương trung bình (validation MSE) ≲ 0.02 tính theo đơn vị hành động, cộng với lượt chạy thử có kết nối máy tính ở bước tiếp theo. Tính năng đánh giá vòng kín trong mô phỏng đã được gỡ bỏ khỏi `distill.py` vì nó không mang lại nhiều ý nghĩa đối với giáo viên đã tinh chỉnh trên thiết bị thực — chúng thường không vượt qua bài kiểm tra mô phỏng dù cân bằng rất tốt trên phần cứng thực tế.

## 5. Kiểm tra chính sách học sinh (student) trên thiết bị — Có kết nối máy tính

Xác nhận xem học sinh có tái hiện trung thực hành vi của giáo viên hay không *trước khi* nạp trực tiếp lên Nano. Vẫn sử dụng kịch bản `run_policy.py` tương tự nhưng trỏ tới tệp `.pt` của học sinh. Mất khoảng 30 giây chạy thiết bị:

```bash
python run_policy.py \
    --policy runs/async_v1_extend/distill_h32_aug/student.pt \
    --port /dev/cu.usbserial-1130 \
    --duration-s 30
```

Yêu cầu điểm số `upright` proxy nằm trong khoảng **chênh lệch ~0.05** so với điểm của giáo viên ở bước 3. Khoảng cách lớn hơn cho thấy vấn đề lệch phân phối (covariate-shift); hãy tăng cường mô phỏng (`distill.py --sim-augment-steps 200000`), tăng dung lượng của học sinh (mức H=48 vẫn vừa vặn bộ nhớ), hoặc chấp nhận kết quả hiện tại.

## 6. Nạp sketch chạy độc lập — Loại bỏ kết nối máy tính

```bash
# Xuất các trọng số PROGMEM vào thư mục sketch của Arduino
python export_weights.py \
    --student runs/async_v1_extend/distill_h32_aug/student.pt \
    --header  ../../../RotaryInvertedPendulum-arduino/RLControl/policy_weights.h \
    --source-name async_v1_extend/distill_h32_aug

# (Tùy chọn nhưng khuyến nghị) cập nhật các giá trị tham chiếu tự kiểm tra (self-test) lúc khởi động
# trong tệp RLControl.ino để khớp với học sinh mới. Tính toán bằng cách chạy lệnh:
#   python -c "import torch, numpy as np; from distill import StudentMLP, _student_predict_factory; \
#       ckpt = torch.load('runs/async_v1_extend/distill_h32_aug/student.pt', \
#           map_location='cpu', weights_only=True); \
#       m = StudentMLP(hidden=ckpt['hidden'], obs_dim=ckpt['obs_dim'], act_dim=ckpt['act_dim']); \
#       m.load_state_dict(ckpt['state_dict']); pred = _student_predict_factory(m); \
#       print('hanging:', float(pred(np.array([0,0,-1,0,0],dtype=np.float32))[0])); \
#       print('upright:', float(pred(np.array([0,0,1,0,0],dtype=np.float32))[0]))"

# Nạp chương trình
cd ../../..
arduino-cli compile --upload -p /dev/cu.usbserial-1130 \
    --fqbn arduino:avr:nano:cpu=atmega328 \
    RotaryInvertedPendulum-arduino/RLControl
```

Hãy giữ con lắc treo thẳng đứng xuống phía dưới tại thời điểm **3 giây trễ lúc khởi động** kết thúc (khi đèn LED chuyển từ nhấp nháy chậm sang nhấp nháy nhanh) — tư thế đó sẽ trở thành điểm 0 của encoder cho quá trình điều khiển.

Xác nhận trên Serial Monitor ở tốc độ 500 kbaud:
- Lúc khởi động sẽ in `[boot] policy(hanging) = X.XXXX` và `[boot] policy(upright) = X.XXXX`. Các giá trị này phải khớp với các giá trị bạn đã tính toán ở lệnh phụ trợ phía trên với sai số ≤ 1e-3 (so sánh số thực AVR float và PyTorch). Nếu không khớp, lượt truyền xuôi C++ (forward pass) hoặc việc truy cập bộ nhớ PROGMEM đang bị lỗi — hãy xuất lại trọng số, biên dịch lại và nạp lại.
- Khi đã bắt đầu điều khiển: con lắc sẽ tự swing-up dựng thẳng và cân bằng trong vòng ~3 giây, giữ yên ≥ 30 giây khi không bị tác động, và có khả năng tự phục hồi khi bị gõ nhẹ gây mất cân bằng.

## Chạy lại các bước riêng lẻ

Mỗi bước đều có tính không đổi (idempotent) và có thể được chạy độc lập:

| Mục tiêu | Chạy lại | Bắt đầu từ |
|---|---|---|
| Điều chỉnh phần thưởng hoặc phạm vi DR | bước 1 | ban đầu |
| Thêm dữ liệu chạy thực tế | bước 2 | `--resume-buffer` |
| Thử học sinh lớn hơn / nhỏ hơn | bước 4 | giáo viên hiện có |
| Nạp lại chương trình với cùng học sinh | bước 6 | tệp `.h` hiện có |

## Khắc phục sự cố

- **Giáo viên cân bằng được khi cắm máy tính nhưng học sinh thất bại trên Nano**: Lỗi dễ xảy ra nhất là căn chỉnh điểm 0 của encoder — giá trị được ghi nhận *tại thời điểm bắt đầu* trong tệp `RLControl.ino`, nhưng nếu sketch được nạp khi con lắc không ở tư thế treo thẳng và không được định vị lại trong 3 giây trễ, hệ tọa độ của chính sách sẽ bị xoay lệch. Hãy reset Arduino khi con lắc đang treo thẳng đứng xuống dưới.
- **Tự kiểm tra lúc khởi động in `[FATAL] FastAccelStepper config rejected`**: Tốc độ `MOTOR_MAX_SPEED` yêu cầu vượt quá giới hạn tối đa AVR của thư viện FastAccelStepper là 50 kSteps/s cho một động cơ bước duy nhất. Hãy kiểm tra hằng số trong tệp `RLControl.ino`.
- **Con lắc dao động nhưng không bao giờ dựng thẳng đứng được**: Lực tác động của động cơ quá thấp. Hãy xác nhận xem hằng số `MOTOR_ACCELERATION` có khớp nhau giữa tệp `LowLevelServer.ino` (sử dụng khi tinh chỉnh) và tệp `RLControl.ino` (sử dụng khi triển khai chạy độc lập) hay không. Chúng bắt buộc phải khớp nhau, nếu không chính sách sẽ được huấn luyện trên một tập động lực học này nhưng lại được triển khai trên một tập động lực học khác.
