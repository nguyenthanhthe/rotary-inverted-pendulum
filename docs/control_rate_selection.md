# Lựa chọn tần số điều khiển (và `max_action_delta_rad`)

Việc lựa chọn các thông số này **không** phải là một lựa chọn ngẫu nhiên. Nếu chọn sai, chính sách (policy) sẽ hoạt động tốt trong mô phỏng (sim) nhưng lại bị lỗi trên phần cứng thực tế, hoặc ngược lại. Khi được thực hiện đúng cách, lựa chọn này sẽ được rút ra trực tiếp từ các phép đo nhận dạng hệ thống (sysid).

Tài liệu này trình bày về *cách thực hiện* (công thức tính toán số) và *tại sao* (các ràng buộc vật lý). Để biết *cấu trúc của một bước chuyển đổi (transition)* sau khi các thông số này được cố định, xem [`rl_transitions.md`](rl_transitions.md). Để biết *cách* áp đặt tần số điều khiển đã chọn trong thời gian chạy (runtime), xem [`async_control_architecture.md`](async_control_architecture.md).

## Phân loại nút tinh chỉnh: tự động tính toán (auto-derived) so với người dùng tự đặt (user-set)

Pipeline RL có nhiều tham số có thể điều chỉnh. Chúng được chia rõ ràng thành hai danh mục và sự phân chia này được áp đặt trực tiếp trong mã nguồn:

**Tự động tính toán từ các đầu vào khác** (bạn không thiết lập trực tiếp các thông số này):

| Nút tinh chỉnh | Được tính toán từ | Tại sao tự động |
|---|---|---|
| `vel_filter_cutoff_hz` | `control_freq_hz` | Đảm bảo tính nhất quán — giữ tần số cắt của bộ lọc dưới tần số Nyquist và trên băng thông tín hiệu (signal bw). Không cần lựa chọn thiết kế. |
| Phạm vi trễ DR trong chương trình học (steps) | `control_freq_hz` × phạm vi trễ vật lý (ms) | Độ trễ vật lý (ms) là đơn vị có ý nghĩa; số lượng bước (step count) chỉ là để quản lý sổ sách. |
| `n_substeps` (mô phỏng) | `control_freq_hz` / physics_dt | Phục vụ quản lý trong MuJoCo, không có nội dung thiết kế. |

**Người dùng tự đặt** (bạn quyết định cụ thể các thông số này):

| Nút tinh chỉnh | Ý nghĩa thiết kế | Tại sao cần đặt cụ thể |
|---|---|---|
| `control_freq_hz` | Lựa chọn cửa sổ tần số điều khiển từ sysid | Quyết định thiết kế: điểm rơi trong khoảng `[5×f_n, 3×BW_motor]` |
| `max_action_delta_rad` | Giới hạn thay đổi vị trí (slew budget) mỗi tick | Quyết định thiết kế: chính sách cần phản ứng nhanh đến mức nào? |
| `dt_jitter_frac` (cho mỗi giai đoạn) | Cường độ DR tác động lên thời gian | Quyết định chương trình học: mức độ chính quy hóa (regularization) cần mạnh mẽ thế nào? |
| `k_action`, `k_θ`, `k_motor_pos`, v.v. | Trọng số định hình chính sách (reward weights) | Quyết định thiết kế: hành vi nào sẽ được phần thưởng (reward) ưu tiên? |
| Phạm vi trễ của các giai đoạn (bằng ms, trước khi nhân tỉ lệ) | Độ rộng DR cho mỗi giai đoạn | Quyết định chương trình học: độ khó của mỗi giai đoạn thế nào? |

**Quy tắc**: tự động tính toán khi có một câu trả lời đúng hiển nhiên được rút ra một cách cơ học từ một đầu vào khác. Hãy để người dùng tự đặt khi lựa chọn đó thể hiện *ý đồ thiết kế* về cách chính sách nên hoạt động. Tự động tính toán là để đảm bảo tính nhất quán; nút tinh chỉnh do người dùng tự đặt là thiết kế.

Sự phân chia này có nghĩa là:
- Người dùng mới chạy thiết bị này không cần nhớ công thức bộ lọc — chỉ cần chọn tần số điều khiển và phần còn lại sẽ tự động được tính toán một cách hợp lý.
- Vẫn có các đường dẫn ghi đè (override paths) cho mọi nút tự động tính toán (chỉ cần truyền vào một giá trị cụ thể), vì vậy không có gì ngăn cản việc thử nghiệm.
- Các nút do người dùng tự đặt mới thực sự là bề mặt thiết kế — tập hợp nhỏ các con số mà bạn phải suy nghĩ.

## Các đầu vào (từ sysid)

Hai con số vật lý, cả hai đều được tạo ra bởi Phase 0 sysid:

| Đại lượng | Cách đo lường | Giá trị của thiết bị này |
|---|---|---|
| **Băng thông động cơ** `BW_motor` | `1 / (2π × rise_time_95)` từ phản hồi bước (step response) | **16 Hz** (thời gian tăng 64 ms) |
| **Tần số tự nhiên của con lắc** `f_n` | `1 / period_s` từ khớp quay tự do (free-swing fit) | **1.9 Hz** (chu kỳ 0.526 s) |

Cả hai đều được lưu trong tệp `sysid_params.json` sau khi thực hiện quy trình runbook. Hãy đo và tính toán lại chúng bất cứ khi nào thiết bị có thay đổi (thay vòng bi mới, thay động cơ, v.v.). Xem [`sysid_runbook.md`](sysid_runbook.md) để biết quy trình đo lường.

## Cửa sổ tần số điều khiển hợp lệ

Tần số lấy mẫu phải thỏa mãn đồng thời cả hai bất đẳng thức sau:

1. **Giới hạn dưới — do con lắc quyết định**: lấy mẫu đủ nhanh để điều khiển chế độ không ổn định thẳng đứng của con lắc. Quy tắc: `f_ctrl ≥ 5–10 × f_n`. Đối với thiết bị này, giá trị tối thiểu là ≥ 10–20 Hz.
2. **Giới hạn trên — do động cơ quyết định**: không lấy mẫu nhanh hơn khả năng đáp ứng vật lý của động cơ. Quy tắc: `f_ctrl ≤ 2–3 × BW_motor`. Đối với thiết bị này, giá trị tối đa là ≤ 32–50 Hz.

Đối với thiết bị này: **cửa sổ tần số nằm trong khoảng 30–50 Hz.** Nếu nằm ngoài khoảng này:

- Dưới 30 Hz: con lắc có thể dao động đáng kể giữa các tick; điều khiển vòng kín (closed-loop control) sẽ bị thiếu giảm chấn (underdamped) hoặc mất ổn định.
- Trên 50 Hz: động cơ sẽ lọc thông thấp (low-pass) các mục tiêu gửi đến; chính sách sẽ tự chiến đấu với các lệnh cũ chưa kịp đáp ứng của chính nó. Thực tế quan sát được: khi huấn luyện và triển khai ở tần số 100 Hz, giá trị trung bình đại diện cho trạng thái thẳng đứng của chính sách là 0.69. Ở tần số 35 Hz: đạt 0.89. Cùng kiến trúc mô hình; cùng một phần cứng. Sự không khớp tần số đơn thuần đã giải thích cho khoảng cách này.

## Tốc độ thay đổi (Slew rate) — ngân sách cho mỗi giây

*Tốc độ thay đổi (slew rate)* là tốc độ mà chính sách được phép thay đổi vị trí mục tiêu của động cơ, đo bằng rad/s. Nó là tích của hai nút tinh chỉnh:

```
slew = max_action_delta_rad × control_freq_hz   (rad/s)
```

**Ý nghĩa trực quan**: nếu chính sách của bạn xuất ra hành động tối đa (`a = +1.0`) ở mỗi tick trong vòng một giây, đây là khoảng cách mà mục tiêu động cơ di chuyển trong giây đó. Đây là yêu cầu tối đa mà chính sách có thể đặt ra cho cơ cấu chấp hành (actuator) trong một giây duy trì các lệnh cực đoan liên tục.

**Giới hạn hợp lý**: `slew ≤ BW_motor × A_max`, trong đó `A_max ≈ 0.2 rad` là bước đặt mục tiêu đơn lẻ lớn nhất mà động cơ bước có thể thực hiện hiệu quả mà không bị chi phối bởi biên dạng gia tốc hình thang (trapezoidal profile). Đối với thiết bị này:

`slew ≤ 16 Hz × 0.2 rad = 3.2 rad/s` (mức an toàn)

Trong thực tế, tốc độ lên đến ~5 rad/s vẫn ổn (khoảng dự phòng băng thông động cơ và trễ bậc một không bị sụt giảm quá đột ngột).

## Cách kết hợp — các con số của thiết bị này

| Cấu hình | tần số (rate) | delta | slew | Trong cửa sổ? | Kết quả thực tế |
|---|---|---|---|---|---|
| Mặc định mô phỏng (Phase 1) | 100 Hz | 0.10 | 10 rad/s | tần số quá cao, slew quá cao | 0.69 thẳng đứng; rung giật, động cơ tự triệt tiêu |
| 50 Hz | 50 Hz | 0.10 | 5.0 rad/s | tần số ở sát biên trên | 0.74 thẳng đứng; điểm thu hút "hiệu chỉnh tích cực" |
| 40 Hz | 40 Hz | 0.10 | 4.0 rad/s | trong cửa sổ hợp lệ | 0.72 thẳng đứng; cùng điểm thu hút tích cực |
| **35 Hz** | **35 Hz** | **0.10** | **3.5 rad/s** | **thiên về điểm tĩnh** | **0.91 thẳng đứng; điểm thu hút "hành động tối thiểu" — động cơ hầu như không chuyển động khi đã cân bằng** |

## Hai điểm thu hút (attractors) — quan sát thực tế

SAC hội tụ một cách đáng tin cậy vào một trong hai chính sách có chất lượng khác nhau hoàn toàn trên thiết bị này, tùy thuộc vào ngân sách slew rate:

- **Điểm thu hút tĩnh (Calm attractor) (slew ≤ ~3.5 rad/s)**: các lệnh động cơ nằm trong khoảng ±0.05 rad ngay cả khi đang cân bằng hoàn toàn. Chi phí hành động cho mỗi bước tiến dần về 0. Trực quan: con lắc đứng yên như tượng, động cơ về cơ bản không chuyển động. Chống nhiễu tốt với tiếng ồn của vòng bi; chính sách coi thiết bị là "đủ tự ổn định" khi ở gần vị trí thẳng đứng.
- **Điểm thu hút hiệu chỉnh tích cực (Active-correction attractor) (slew ≥ ~4.0 rad/s)**: các lệnh động cơ thường xuyên dao động ±0.5 rad ngay cả khi đã cân bằng. Chi phí góc lệch (theta) cho mỗi bước vẫn tốt (chính sách giữ θ gần bằng 0), nhưng chi phí *hành động* (action) cho mỗi bước rất cao. Trực quan: con lắc bị rung giật khi động cơ liên tục di chuyển qua lại.

Ranh giới giữa chúng rất rõ ràng — **giữa 35 và 40 Hz trên thiết bị này**, với `max_action_delta_rad=0.10`. Điều này được xác nhận bằng cách huấn luyện một chính sách 40 Hz bắt đầu từ checkpoint 35 Hz tĩnh: 50 tập tinh chỉnh ở tần số cao hơn đã lật nó sang điểm thu hút tích cực.

Các trọng số phần thưởng (`k_action=0.05`, `k_θ=1.0`) chưa phạt đủ mạnh đối với nỗ lực hành động so với độ lệch θ. SAC sẽ chọn hiệu chỉnh tích cực bất cứ khi nào nó có đủ ngân sách slew rate cho việc đó; điều duy nhất giữ chính sách 35 Hz ở điểm thu hút tĩnh là do mức 3.5 rad/s không *đủ* slew rate để hiệu chỉnh tích cực mang lại phần thưởng cao hơn so với ổn định tĩnh thụ động.

## Quy trình lựa chọn tần số điều khiển (rate) và delta cho thiết bị mới

1. Chạy sysid (xem [`sysid_runbook.md`](sysid_runbook.md)). Ghi lại `BW_motor` và `f_n`.
2. Tính toán cửa sổ tần số: `[5 × f_n, 3 × BW_motor]`. Nếu khoảng này rỗng, bạn cần một động cơ nhanh hơn trước khi thiết bị này có thể điều khiển được.
3. Chọn một tần số trong cửa sổ đó. **Ưu tiên thiên về phía biên dưới.** Theo thực nghiệm (xem phân tích điểm thu hút ở trên), SAC có xu hướng rơi vào điểm thu hút "tĩnh" với hành động tối thiểu khi slew rate bằng hoặc thấp hơn `BW_motor × 0.2`, và rơi vào điểm thu hút "nhạy" hiệu chỉnh tích cực khi ở trên mức đó. Tần số ở biên dưới cũng có khoảng dự phòng chống lại sự thay đổi băng thông động cơ dưới tải nặng. Trực giác ngây thơ cho rằng "tần số cao hơn = phản ứng nhanh hơn" là sai lầm ở đây, vì khả năng phản ứng của chính sách đến từ phân phối hành động của nó chứ không phải tần số đưa ra quyết định — và điểm thu hút tích cực tạo ra phản ứng *ít* hữu ích hơn (chỉ là tự triệt tiêu lẫn nhau).
4. Chọn `max_action_delta_rad` sao cho `delta × rate ≤ BW_motor × 0.2 ≈ ~3.2-3.5 rad/s` (để giữ hệ thống ở điểm thu hút tĩnh).
5. Cấu hình `control_freq_hz` và `max_action_delta_rad` nhất quán trên **quá trình huấn luyện mô phỏng (sim training)**, **tinh chỉnh (fine-tuning)**, **và triển khai thực tế (deployment)**. Chính sách học theo tần số mà nó được huấn luyện; việc triển khai không khớp tần số chính là lỗi mà chúng tôi xây dựng `async_control.py` để ngăn chặn — xem [`async_control_architecture.md`](async_control_architecture.md).

## Tóm tắt trình tự

```
sysid → BW_motor + f_n
      → cửa sổ tần số [5·f_n, 3·BW_motor]
      → chọn f_ctrl trong cửa sổ (thấp hơn = an toàn, cao hơn = phản ứng nhanh)
      → chọn max_action_delta_rad sao cho delta × f_ctrl ≤ ~3.5 rad/s
      → thiết lập các giá trị đó một lần; sử dụng ở mọi nơi (sim, fine-tune, deploy)
```

## Tần số cắt của bộ lọc vận tốc — lựa chọn từ tần số điều khiển

Quy trình quan sát lấy hiệu sai phân (finite-differences) vị trí thô của cảm biến mã hóa vòng quay và đưa kết quả qua bộ lọc thông thấp IIR bậc 1. Tần số cắt `vel_filter_cutoff_hz` nên được chọn tương đối theo tần số điều khiển đã chọn, không chọn tuyệt đối. Nếu bạn thay đổi `control_freq_hz`, hãy tính toán lại tần số cắt.

### Ràng buộc

Hai bất đẳng thức, tương tự như cửa sổ tần số điều khiển:

1. **Giới hạn dưới — bảo toàn tín hiệu.** Tần số cắt phải nằm *trên* tần số có nghĩa cao nhất trong động lực học vòng kín của bạn. Đối với con lắc ngược, đó là hằng số thời gian không ổn định thẳng đứng `1/τ = ω_n ≈ 12 Hz` đối với thiết bị này. Dưới mức cắt ~12 Hz, bộ lọc sẽ làm suy hao chính tín hiệu mà chính sách cần phản ứng.
2. **Giới hạn trên — tần số Nyquist.** Độ dốc suy giảm của bộ lọc bậc 1 khá nhẹ nhàng. Tần số cắt bằng hoặc trên `0.5 × control_freq_hz` (Nyquist) có nghĩa là mỗi mẫu mới đóng góp ≥75% vào ước tính đang chạy — bộ lọc về cơ bản chỉ là một đường truyền thẳng (passthrough) và không thực hiện làm mịn hữu ích.

Điểm tối ưu: **tần số cắt nằm giữa tần số tín hiệu cao nhất và khoảng một nửa tần số Nyquist**.

### Các con số của thiết bị này

Chế độ không ổn định thẳng đứng của con lắc ở khoảng ~12 Hz, vì vậy tần số cắt ≥ ~10 Hz để bảo toàn tín hiệu. Tần số Nyquist ở tần số điều khiển đã chọn sẽ quyết định giới hạn trên. Phổ nhiễu của encoder giảm dần ở đâu đó quanh mức ~20 Hz, do đó giới hạn ở đó để tránh việc bộ lọc trở thành truyền thẳng ở tần số điều khiển cao.

### Tự động tính toán (mặc định hiện tại)

Cả `real_env.py` và `run_policy.py` đều tự động tính toán tần số cắt từ `control_freq_hz` nếu bạn không truyền tham số `--vel-filter-cutoff-hz`:

```
cutoff = min(20.0, max(10.0, 0.4 × control_freq_hz))
```

| Tần số điều khiển | Tần số cắt tự động | Tại sao |
|---|---|---|
| ≥ 50 Hz | 20.0 (giới hạn trên) | Trên mức này bộ lọc sẽ gần như truyền thẳng; giới hạn ở 20 để giữ chức năng lọc nhiễu thực tế |
| 40 Hz | 16.0 | 40% tần số điều khiển, ~80% tần số Nyquist |
| 35 Hz | 14.0 | 40% tần số điều khiển |
| **30 Hz** | **12.0** | 40% tần số điều khiển, ngay trên biên tín hiệu 12 Hz |
| ≤ 25 Hz | 10.0 (giới hạn dưới) | Giới hạn dưới bảo vệ sự bảo toàn tín hiệu; dưới mức này bộ lọc bắt đầu làm suy hao động lực học hữu ích của con lắc |

Hệ số nhân 0.4 mang lại khoảng ~80% tần số Nyquist khi không bị giới hạn — thực hiện lọc tốt, không phải truyền thẳng. Các giới hạn 10/20 Hz phản ánh băng thông tín hiệu (~12 Hz) và phổ nhiễu (~20 Hz) của *thiết bị này*. Trên một thiết bị khác, các giá trị này sẽ thay đổi theo kết quả sysid mới.

### Khi nào cần ghi đè (override)

Tham số `--vel-filter-cutoff-hz N` bắt buộc một giá trị cụ thể. Các lý do để ghi đè:

- **Bạn thấy các hành động bị nhiễu/rung khi triển khai**: tần số cắt có thể quá cao. Hãy giảm ~2-3 Hz so với giá trị tự động.
- **Bạn thấy phản ứng chậm chạp với các tác động nhiễu**: tần số cắt quá thấp. Hãy tăng ~2-3 Hz so với giá trị tự động.
- **Bạn đang thay thế con lắc / động cơ**: hãy đo lại băng thông tín hiệu và phổ nhiễu của thiết bị từ sysid; chọn các giá trị giới hạn mới trước khi tin cậy vào tính năng tự động tính toán.

### Khi nào cần chỉnh lại (retune)

Thay đổi tần số cắt nếu xảy ra bất kỳ điều nào sau đây:

- **Tần số điều khiển thay đổi đáng kể** (nhiều hơn ~2 lần): Tần số Nyquist dịch chuyển; hãy chọn một tần số cắt mới trong điểm tối ưu mới.
- **Động lực học của thiết bị thay đổi** (con lắc khác, động cơ khác): tần số không ổn định thẳng đứng dịch chuyển. Hãy đo lại qua sysid; tính toán lại giới hạn dưới.
- **Bạn thấy các hành động bị nhiễu khi triển khai**: tần số cắt quá cao, việc giảm tần số cắt xuống ~12 Hz giúp làm mịn hơn.
- **Bạn thấy phản ứng chậm chạp với các tác động nhiễu**: tần số cắt quá thấp, làm suy hao tín hiệu hữu ích. Hãy nâng lên mức 18-20 Hz.

## Vị trí lưu trữ các giá trị đã chọn trong mã nguồn

| Nút tinh chỉnh | Mặc định | Nơi thiết lập / đọc |
|---|---|---|
| `control_freq_hz` | **35** ở mọi nơi — `pendulum_env.py`, `real_env.py`, `async_control.py`, và mặc định của `--control-freq` trong `train_sac.py` / `finetune_async.py` / `eval_randomized.py` / `run_policy.py` / `distill.py`. Giá trị mặc định lịch sử 100 Hz có trước phát hiện thực nghiệm 35 Hz (Phase 4.6) và đã sai đối với thiết bị này. | Khởi tạo env mô phỏng `__init__`, khởi tạo env thực tế `__init__`, vòng lặp triển khai |
| `max_action_delta_rad` | 0.10 trong khởi tạo env `__init__`; cờ `--max-action-delta-rad` trong `train_sac.py` và `finetune_async.py` | Giới hạn cắt (clipping) bên trong `step()` / `apply_action()` |
| Độ trễ các giai đoạn chương trình học (steps) | Được tính toán trong `curriculum_train.sh` từ mili-giây vật lý × `CONTROL_FREQ` | Phép tính số học Bash ở đầu tập lệnh |
