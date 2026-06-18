# RotaryInvertedPendulum-python

Các công cụ Python cho hệ con lắc ngược quay: điều khiển bằng gamepad, nhận dạng hệ thống (system identification), và pipeline RL (môi trường env, bộ huấn luyện SAC, client triển khai thực tế). Xem tệp [`../RL_PLAN.md`](../RL_PLAN.md) để biết kế hoạch rộng hơn.

## Thiết lập môi trường

Môi trường mamba duy nhất có tên `rotary-inverted-pendulum`, sử dụng Python 3.12. Kênh Conda-forge cung cấp các thư viện khoa học; pip cung cấp MuJoCo / Gymnasium / SB3 / Torch vì chúng được cài đặt đáng tin cậy hơn từ PyPI.

```bash
mamba create -n rotary-inverted-pendulum -y -c conda-forge python=3.12 numpy scipy
mamba activate rotary-inverted-pendulum

# Bản thử nghiệm Gamepad (tập lệnh giao thức văn bản cũ)
mamba install -y pygame pyserial

# Tiện ích bổ sung cho RL + sysid (PyPI)
pip install pyserial 'numpy<2.3' mujoco gymnasium 'stable-baselines3[extra]'
```

Lưu ý:
- Thư viện `numpy<2.3` để giữ tính tương thích với gói `scipy 1.14` cài qua conda. Nếu bạn gặp lỗi không tìm thấy `_ARRAY_API`, có thể một phiên bản `opencv-*` cũ trong thư mục `~/.local/lib/python3.12/site-packages/` đang ghi đè lên cài đặt môi trường. Hãy chạy `pip install --user --upgrade opencv-contrib-python` để cập nhật lại.
- Chỉ dành cho macOS: trình trực quan hóa MuJoCo yêu cầu sử dụng `mjpython` (một trình chạy đặc biệt để xử lý các vấn đề của luồng chính Cocoa). Hãy cài đặt qua cùng lệnh `pip` và gọi `mjpython` thay vì `python` cho bất kỳ tập lệnh nào cần mở cửa sổ trực quan hóa.

## Tần số điều khiển (Control rate)

Tần số hoạt động chuẩn của thiết bị này là **35 Hz** (28.6 ms cho mỗi bước điều khiển, `max_action_delta_rad = 0.10` → tốc độ thay đổi mục tiêu 3.5 rad/s). Đây là điểm hoạt động tối ưu được tìm thấy thực nghiệm trên phần cứng này: tần số 100 Hz đẩy chính sách vào điểm thu hút "hiệu chỉnh tích cực" đầy nhiễu, nơi động cơ dao động ±0.5 rad ngay cả khi đã cân bằng; tần số 30 Hz thiếu lực điều khiển để phục hồi khi bị tác động nhiễu; tần số 35 Hz rơi vào vùng điểm thu hút tĩnh của ranh giới với tốc độ thay đổi mục tiêu ~3.5 rad/s.

Tất cả các cờ lệnh `--control-freq` và giá trị mặc định của tham số khởi tạo `control_freq_hz` trong các tệp `pendulum_env.py`, `real_env.py`, `async_control.py`, `train_sac.py`, `finetune_async.py`, `eval_randomized.py`, `run_policy.py`, và `distill.py` đều mặc định là 35 Hz. Chỉ ghi đè nếu bạn có lý do cụ thể — xem tài liệu [`../docs/control_rate_selection.md`](../docs/control_rate_selection.md) để biết lập luận về cửa sổ tần số hợp lý và dữ liệu thực nghiệm về điểm thu hút tĩnh so với tích cực.

## Quy trình từ đầu đến cuối (End-to-end pipeline)

Để xem hướng dẫn chi tiết từ sysid → huấn luyện mô phỏng → tinh chỉnh thiết bị thực → chưng cất → nạp chương trình với các lệnh chính xác cho từng bước, xem tài liệu [`../docs/end_to_end_runbook.md`](../docs/end_to_end_runbook.md). Mỗi bước đều có tính không đổi (idempotent), vì vậy bạn có thể tham gia vào các giai đoạn riêng lẻ mà không cần chạy lại toàn bộ quy trình.

## Quy trình làm việc (Workflows)

### Điều khiển bằng Gamepad (Legacy)

Điều khiển thông qua firmware giao thức văn bản của Arduino.

```bash
python src/gamepad_control.py
```

### Nhận dạng hệ thống (System identification)

Giai đoạn 0 của kế hoạch RL. Xem tài liệu [`../docs/sysid_runbook.md`](../docs/sysid_runbook.md) để biết quy trình đầy đủ. Khởi động nhanh (khi đã nạp `LowLevelServer.ino`):

```bash
cd src/rl
python sysid_wizard.py                  # toàn bộ quy trình: thu thập + khớp + vẽ đồ thị
python sysid_wizard.py fit --in-dir ... # khớp lại mô hình từ nhật ký có sẵn (không cần thiết bị)
```

Hãy chạy lại sysid bất cứ khi nào thiết bị có thay đổi (lắp ráp lại, vòng bi mới, điều chỉnh động cơ, v.v.) để tệp `sysid_params.json` luôn được cập nhật và trình mô phỏng phản ánh đúng thực tế.

### Huấn luyện RL (Giai đoạn 1+)

Huấn luyện một chính sách SB3 SAC trong môi trường MuJoCo được cấu hình từ tệp `sysid_params.json`. Mặc định chạy một env duy nhất trên CPU; các tùy chọn chạy nhiều env song song (vectorised envs) và chạy trên GPU có sẵn thông qua các cờ lệnh.

```bash
cd src/rl
python train_sac.py --total-steps 500000 --progress-bar
# Xem các đường cong huấn luyện
tensorboard --logdir runs/
```

Tiếp tục huấn luyện / Đánh giá:

```bash
python train_sac.py --resume runs/<run>/last.zip --total-steps 1000000
mjpython train_sac.py --eval runs/<run>/best_model.zip --eval-seconds 30
```

### Huấn luyện theo chương trình học (Curriculum training - Khuyến nghị)

Huấn luyện từ đầu với toàn bộ cấu hình DR mục tiêu (trễ 4-7 bước, v.v.) là một bài toán khó hội tụ cho SAC — chính sách thường bị kẹt từ rất sớm. Trình chạy chương trình học sẽ huấn luyện chính sách qua ba giai đoạn với độ khó DR tăng dần, mỗi giai đoạn tiếp tục (`--resume`) từ tệp `best_model.zip` của giai đoạn trước đó.

```bash
./curriculum_train.sh <run-name-prefix>
# ví dụ đầu ra -> runs/<prefix>_stage1, _stage2, _stage3
```

Chi tiết các giai đoạn (được định nghĩa trong `curriculum_train.sh`):

| Giai đoạn | Dải tau | Dải trễ | Số bước |
|---|---|---|---|
| 1 (dễ) | 0–5 ms | 0–2 bước | 100k |
| 2 (vừa) | 0–10 ms | 2–5 bước | 100k |
| 3 (cuối) | 0–10 ms | 4–7 bước | 100k |

Dải trễ của giai đoạn 3 cuối cùng khớp với độ trễ truyền thông 5 bước đo được trên phần cứng thực tế ở mức cấu hình `MOTOR_ACCELERATION = 50k` và `Vref = 0.485 V`. Ghi đè các tham số của từng giai đoạn qua các biến môi trường (`SEED`, `STEPS_PER_STAGE`, `DEVICE`).

### Đánh giá chính sách (Evaluating a policy)

Hai công cụ bổ trợ cho nhau:

```bash
# Trực quan hóa một lượt chạy thử xác định trong trình xem MuJoCo (sử dụng mjpython trên macOS)
mjpython train_sac.py --eval runs/<run>/best_model.zip --eval-seconds 30

# Kiểm tra khả năng chịu tải của chính sách trên N=20 tập có cấu hình vật lý ngẫu nhiên.
# Báo cáo kết quả ✓/✗ cho từng tập, tỷ lệ thành công "solved", và mẫu DR (tau, trễ)
# được sử dụng cho mỗi tập. Xem chú thích trong tập lệnh để biết định nghĩa đầy đủ về tiêu chí thành công.
python eval_randomized.py runs/<run>/best_model.zip --n-episodes 20
```

### Nhật ký quỹ đạo + Phân tích sim-to-real

Lệnh `run_policy.py --log <path>.npz` lưu quỹ đạo ở cấp độ bước của lượt chạy phần cứng thực tế. Hai tập lệnh bổ trợ sau đó sẽ phân tích nhật ký đó để khớp mô hình sysid tinh chỉnh đối với động lực học điều khiển thực tế (thông tin này thường hữu ích hơn so với các phép đo sysid kiểm tra động cơ đơn thuần):

```bash
# Khớp thô độ trễ truyền thông hiệu dụng và hằng số tau bậc một
# dựa trên dữ liệu motor_target → motor_actual.
python analyze_run.py /tmp/policy_run.npz

# Chạy lại cùng chuỗi hành động đã ghi nhận trong môi trường mô phỏng và báo cáo
# sai số quỹ đạo của động cơ / con lắc so với thiết bị thực. Cho bạn thấy
# khoảng cách sim-to-real thực tế lớn thế nào trên một lượt chạy thực tế.
python sim_vs_real.py /tmp/policy_run.npz
```

### Triển khai trên phần cứng thực tế (Giai đoạn 3)

Chạy giao thức nhị phân của `LowLevelServer.ino` trên Arduino. Hãy nạp chương trình `LowLevelServer` cho Arduino (sysid sử dụng một chương trình khác). Luôn chạy thử với tùy chọn `--dry-run` trước để xác thực giao thức; sau đó chạy thực tế với sự giám sát chặt chẽ thiết bị.

```bash
cd src/rl
python run_policy.py --policy runs/<run>/best_model.zip \
    --port /dev/cu.usbserial-1130 --duration-s 5 --dry-run
python run_policy.py --policy runs/<run>/best_model.zip \
    --port /dev/cu.usbserial-1130 --duration-s 5
```

Client triển khai sẽ cắt vị trí lệnh trong khoảng ±125° (nằm trong giới hạn cơ khí ±135°) và ngắt động cơ khi thoát chương trình hoặc khi nhận được tín hiệu SIGTERM / SIGINT.

## Bố cục mã nguồn

```
src/
  gamepad_control.py        # điều khiển bằng gamepad qua giao thức văn bản cũ
  rl/
    pendulum_env.py         # Môi trường Gymnasium (MuJoCo) cấu hình từ sysid_params.json
    train_sac.py            # Trình huấn luyện SB3 SAC với các callback lưu và đánh giá mô hình
    curriculum_train.sh     # Trình chạy chương trình học 3 giai đoạn (dễ -> đầy đủ DR)
    eval_randomized.py      # Kiểm tra khả năng chịu tải trên N tập ngẫu nhiên (tỷ lệ thành công)
    run_policy.py           # Client triển khai thực tế Giai đoạn 3 (giao thức nhị phân)
    analyze_run.py          # Tính toán hằng số tau / độ trễ hiệu dụng từ nhật ký run_policy
    sim_vs_real.py          # Chạy lại chuỗi hành động đã ghi nhận trong mô phỏng để so sánh
    lowlevel_client.py      # Client Python để giao tiếp với LowLevelServer.ino
    sysid_wizard.py         # Trình hướng dẫn sysid tương tác (thu thập/khớp/xác thực động cơ)
    sysid_core.py           # Tính toán toán học cho sysid (khớp đường bao + tính tham số)
    sysid_params.json       # Đầu ra của sysid_wizard.py (được commit vào git)
    freeswing_probe.py      # Bộ xác thực độc lập so sánh mô phỏng và thực tế
    sysid_runs/             # Các bản ghi của từng phiên (được đưa vào gitignore)
    runs/                   # Các sản phẩm huấn luyện (được đưa vào gitignore)
test/
  serial_test.py            # kiểm tra kết nối serial cơ bản
  gamepad_test.py           # kiểm tra hoạt động của gamepad
```
