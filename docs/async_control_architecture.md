# Kiến trúc điều khiển bất đồng bộ (Async Control Architecture)

Tài liệu này giải thích cách `finetune_async.py` giữ vòng lặp điều khiển của thiết bị (rig) ở tần số nghiêm ngặt 100 Hz (hoặc bất kỳ tần số cấu hình nào) trong khi các cập nhật gradient của SAC chạy song song. Đây là tài liệu về hệ thống runtime — nó không bàn về ngữ nghĩa RL (xem [`rl_transitions.md`](rl_transitions.md)) hay cách *chọn* tần số điều khiển (xem [`control_rate_selection.md`](control_rate_selection.md)). Nó chỉ giải thích cách chúng tôi *áp đặt* tần số đã chọn đó.

## Lỗi (bug) mà kiến trúc này ngăn chặn

Trong quá trình tinh chỉnh (fine-tuning) ở Giai đoạn 4, chúng tôi phát hiện ra rằng hàm `SAC.learn()` của thư viện SB3 chạy tuần tự hai hàm `collect_rollouts` (gọi `env.step()`) và `train()` (cập nhật gradient) trong **một thread duy nhất**. Với tham số `--gradient-steps 4`, các cập nhật gradient mất khoảng ~20 ms cho mỗi bước môi trường (env step). Logic điều phối nhịp độ của env (`time.sleep(next_tick - time.monotonic())`) đã âm thầm kéo tần số cấu hình từ 100 Hz xuống chỉ còn ~35 Hz thực tế. Chính sách đã học theo *động lực học ở tần số 35 Hz*. Khi triển khai thực tế ở tần số 100 Hz, động cơ bị rung giật (chatter) và trượt bước (step-skipping) — cùng một bộ trọng số (weights) nhưng hành vi thực tế quan sát được hoàn toàn khác nhau.

`finetune_async.py` là giải pháp sửa đổi kiến trúc: một vòng lặp huấn luyện tùy chỉnh (thay thế cho `model.learn()`) quản lý hai thread hợp tác với nhau.

## Hai thread

```
┌──────────────────────────────────────────┐    ┌──────────────────────────────────┐
│ Control thread (background, strict 100Hz)│    │ Learner thread (main, best effort)│
│                                          │    │                                  │
│ mỗi 10 ms:                               │    │ vòng lặp:                        │
│   action ← snapshot.predict(obs)         │    │   transitions ← queue.drain()    │
│   env.apply_action(action)               │    │   for t: replay_buffer.add(t)    │
│   sleep_busywait_until_next_tick()       │ →→ │   nếu qua giai đoạn warmup:      │
│   next_obs ← env.observe_and_step(...)   │    │     model.train(K, batch_size)   │
│   queue.put(transition)                  │    │     snapshot.refresh_from(actor) │
│                                          │    │                                  │
└──────────────────────────────────────────┘    └──────────────────────────────────┘
            │                                                 │
            ↓                                                 ↓
        TransitionQueue (deque + lock) — nhà sản xuất/tiêu thụ giữa các thread
                                                               │
                                                               ↓
                                               model.replay_buffer (SB3 ReplayBuffer)
```

Control thread (luồng điều khiển) **không bao giờ bị chặn (block) bởi learner**. Learner (luồng huấn luyện) **không bao giờ bị chặn bởi thiết bị**. Chúng giao tiếp thông qua một queue chung và một replay buffer chung, cả hai đều được bảo vệ bằng khóa (lock).

## Ba khóa (locks)

1. **`LowLevelClient._lock`** (cho mỗi phiên giao dịch serial). Ngăn chặn các byte dữ liệu bị xen lẫn khi control thread đang ở giữa quá trình gọi `get_state` và main thread (luồng chính) kích hoạt `disengage_motor` vì lý do an toàn.

2. **`replay_buffer_lock`** (do bộ điều phối sở hữu). Được giữ trong quá trình gọi `replay_buffer.add()` *và* trong suốt cuộc gọi `model.train(K, batch)`. Hàm `train()` của SAC gọi `sample()` K lần bên trong; chúng ta muốn có một chế độ xem tĩnh (frozen view) của buffer qua các cuộc gọi đó. Trên thực tế, sự tranh chấp (contention) bằng không — cả hai bên tranh chấp đều nằm trên main thread; khóa này tồn tại để dự phòng cho tương lai và phục vụ việc kiểm tra vết (audit trail).

3. **`PolicySnapshot._lock`**. Bảo vệ `self._net.load_state_dict(...)` (được gọi bởi learner sau hàm `train()`) khỏi việc tranh chấp với `self._net(obs)` (được gọi bởi control thread). **Bắt buộc**: Hàm `optimizer.step()` của PyTorch thay đổi tham số `.data` trực tiếp (in place), và một lượt lan truyền xuôi (forward pass) chạy đồng thời với ghi đè dữ liệu đó sẽ tạo ra một hành động bị lỗi. Snapshot (ảnh chụp nhanh) là một bản copy sâu (deepcopy) của `model.actor`; đối tượng actor làm việc của learner không bị ảnh hưởng bởi control thread.

## Tại sao lại dùng snapshot thay vì actor trực tiếp?

Nếu control thread gọi trực tiếp `model.actor(obs)`, mỗi lượt suy luận (inference) sẽ tranh chấp với bất kỳ bước nào của `optimizer.step()` đang chạy trên learner. Đọc các tham số được cập nhật một phần sẽ tạo ra hành động bị lỗi — điều này có thể chấp nhận được trong một thử nghiệm benchmark, nhưng rất nguy hiểm khi điều khiển động cơ thực tế. Snapshot thêm một chi phí sao chép `state_dict` cho mỗi chu kỳ huấn luyện (mất vài micro giây) nhưng mang lại kết quả đọc nhất quán.

## Điều phối nhịp độ — chi tiết về busy-wait

Hàm `time.sleep` trên macOS thường vượt quá mục tiêu khoảng 1-2 ms (do không có `SCHED_FIFO`). Cách ngủ toàn bộ thời gian còn lại (naive sleep-the-full-remainder) sẽ tạo ra tần số trung bình khoảng ~83 Hz khi bạn yêu cầu 100 Hz. Cách khắc phục gồm hai giai đoạn:

```python
sleep_for = next_tick - time.monotonic()
if sleep_for > 0.001:
    time.sleep(sleep_for - 0.001)
while time.monotonic() < next_tick:
    pass    # chờ bận (busy-wait) trong ≤ 1 ms cuối cùng
```

Cách này chỉ tăng thêm <1% CPU ở tần số 100 Hz, giảm độ rung giật thời gian (jitter) từ ~2 ms xuống dưới <0.1 ms. Các thử nghiệm độc lập V1/V2 đo được thời gian trung bình dt = 10.000 ms, độ lệch chuẩn std < 1 ms.

## Watchdog: quy tắc thời gian 3 lần vi phạm (3-strike)

Bên trong vòng lặp tick, nếu control thread vượt quá thời hạn (deadline) của nó một khoảng lớn hơn `timing_violation_threshold_ms` (mặc định là 5 ms) trong **3 tick liên tiếp**, `AsyncControlLoop` sẽ ngắt kích hoạt động cơ và kích hoạt lỗi `TimingViolation`. Sự khoan dung 3 lần vi phạm này là do bộ điều phối của macOS thỉnh thoảng bị nấc một lần rồi tự phục hồi; vi phạm 3 lần liên tiếp là do lỗi cấu trúc (có thể luồng learner đang giữ khóa GIL quá lâu).

Đây là cách người dùng có thể phát hiện lỗi *nhanh chóng* nếu cấu hình sai — không còn hiện tượng giảm tần số âm thầm nữa.

Khi kích hoạt lỗi, vòng lặp cũng gọi `queue.discard_recent(max_strikes - 1)` để các chuyển đổi (transitions) được đưa vào queue trong các tick vi phạm (với dt bị kéo giãn) không làm ô nhiễm replay buffer.

## Xử lý tín hiệu (Signal handling)

Python chỉ gửi các tín hiệu (signals) đến luồng chính (main thread). Bộ điều phối cài đặt các bộ xử lý `SIGINT`/`SIGTERM` để:

1. Đặt cờ chia sẻ `stop_flag` (luồng control thread sẽ kiểm tra cờ này ở đầu mỗi tick → thoát một cách sạch sẽ).
2. Gọi `env.disengage_safely()` (sử dụng `LowLevelClient._lock` để ghi byte disengage một cách an toàn ngay cả khi control thread đang ở giữa lượt đọc).

Hàm đồng bộ `RealRotaryInvertedPendulumEnv.__init__` không còn tự động đăng ký các bộ xử lý tín hiệu nữa — điều đó từng rất quan trọng đối với luồng đồng bộ cũ nhưng lại xung đột với kiến trúc đa luồng mới. Các bộ gọi trực tiếp muốn hành vi cũ có thể tự liên kết bằng `signal.signal(SIGINT, env._on_signal)`.

## Lưu trữ replay buffer (Replay buffer persistence)

`finetune_async.py` lưu tệp `runs/<run-name>/replay_buffer.pkl` (~6 MB cho mỗi 30 nghìn transitions, định dạng pickle) khi kết thúc phiên làm việc và tại mỗi điểm lưu (checkpoint). Cờ `--resume-buffer <path>` cho phép các phiên làm việc tiếp theo tải lại nó thông qua `model.load_replay_buffer`. Điều này có nghĩa là các dữ liệu chuyển trạng thái trên robot thực **được tích lũy qua các phiên làm việc** thay vì bị bỏ đi mỗi lần — điều này rất quan trọng vì dữ liệu robot thực tế đắt hơn khoảng ~1000 lần so với dữ liệu mô phỏng.

Quan trọng: **không** tải các buffer từ các phiên tinh chỉnh đồng bộ cũ (tệp cũ `finetune_real.py`). Những chuyển đổi đó mô tả `(s, a, s')` ở các khoảng thời gian biến đổi từ 28-30 ms (xem "lỗi không khớp tần số" ở trên). Việc trộn chúng với các chuyển đổi tần số nghiêm ngặt mới sẽ dạy cho critic các ánh xạ mâu thuẫn. Cảnh báo tương tự đối với các buffer từ một tần số điều khiển khác — luôn tiếp tục (resume) với cùng tần số điều khiển mà buffer đó được thu thập.

## Giao thức xác thực (Verification protocol)

Được triển khai và chạy theo thứ tự này — mỗi bước chứng minh một đặc tính riêng biệt trước khi kết hợp:

- **V1**: chỉ chạy vòng lặp điều khiển với learner giả lập không làm gì (no-op) → thời gian trung bình dt = 10.000 ms, độ lệch chuẩn std < 1 ms. Chứng minh rằng riêng luồng control thread có thể giữ tần số 100 Hz trên macOS.
- **V2**: vòng lặp điều khiển với learner giả lập chiếm dụng CPU (synthetic CPU-hog) → thời gian giống hệt nhau. Chứng minh rằng learner không làm xáo trộn luồng control thread thông qua khóa GIL.
- **V3** (yêu cầu thiết bị thực): chạy `model.train(K=4)` thực tế với một buffer được tải sẵn → thời gian giống hệt nhau. Chứng minh rằng bước cập nhật PyTorch + numpy + optimizer không làm luồng control thread bị thiếu tài nguyên.
- **V4** (yêu cầu thiết bị thực): tinh chỉnh từ đầu đến cuối ngắn (10 tập × 6 giây, K=4) — tần số tick trung bình do bộ điều phối báo cáo nằm trong khoảng ±2 Hz so với mục tiêu, không có vi phạm nào, chính sách được triển khai lại không bị rung giật ở cùng một tần số.
- **V5** (yêu cầu thiết bị thực): thiết lập lại toàn bộ baseline — chương trình học mô phỏng mới (fresh sim curriculum) ở tần số thiết kế đã chọn, sau đó tinh chỉnh sạch. Chính sách cuối cùng phải duy trì thời gian cân bằng ≥ 5 giây trong các lượt chạy thực tế.

Dữ liệu đo đạc (telemetry) của từng tập được ghi vào `runs/<run-name>/timing.csv`:
`mean_tick_dt_ms`, `std_tick_dt_ms`, `max_overrun_ms`, `n_violations`, `learner_train_calls`, `learner_total_train_s`. Đây là sản phẩm chứng minh chúng ta không bị giảm hiệu năng trong các bản PR sau này.

## Vị trí trong mã nguồn

| Tệp | Chức năng |
|---|---|
| `async_control.py` | Chứa `TransitionQueue`, `PolicySnapshot`, `AsyncControlLoop`. Các nguyên mẫu runtime thuần túy — không có tham chiếu SB3 ngoại trừ việc chấp nhận một `nn.Module` để tạo snapshot. |
| `finetune_async.py` | Bộ điều phối (orchestrator): tải SAC, quản lý các thread + lock, xử lý tín hiệu, lưu trữ buffer + policy. |
| `lowlevel_client.py` | Thêm `_lock` để các cuộc gọi đồng thời `get_state` / `disengage_motor` không bị xen lẫn các byte dữ liệu. |
| `real_env.py` | Tách `apply_action` và `observe_and_step` ra khỏi hàm đồng bộ `step` để vòng lặp bất đồng bộ có thể xen kẽ chúng với các khoảng ngủ được điều phối nhịp độ bên ngoài. |
