# Hướng dẫn nhận dạng hệ thống (System Identification Runbook)

Quy trình từ đầu đến cuối để đo đạc các tham số vật lý của con lắc và tạo ra tệp `sysid_params.json` để sử dụng trong pipeline RL.

Quá trình sysid chạy trên **cùng một** firmware `LowLevelServer.ino` mà chính sách sử dụng khi triển khai — không cần thay đổi firmware, không có rủi ro trôi lệch tham số giữa đo lường và triển khai thực tế.

Trình hướng dẫn (wizard) được chia làm hai giai đoạn:

- **collect (thu thập)** — thu thập dữ liệu trên thiết bị thực tế do người dùng vận hành. Ghi lại các nhật ký `.npz` thô + tệp cấu hình `metadata.json` vào một thư mục được đóng dấu thời gian.
- **fit (khớp mô hình)** — xử lý hậu kỳ thuần túy. Đọc một thư mục chứa các bản ghi, tính toán các tham số, ghi ra tệp `sysid_params.json`, và vẽ đồ thị xác thực so sánh mô phỏng và thực tế (sim-vs-real). Giai đoạn này không cần kết nối với thiết bị thực; có thể chạy lại bao nhiêu lần tùy thích.

Việc khớp lại mô hình (re-fitting) từ các nhật ký đã lưu là vòng lặp lặp nhanh khi cải tiến thuật toán toán học. Việc thu thập lại dữ liệu (re-collecting) trên thiết bị chỉ cần thiết khi thay đổi tần số đọc cảm biến, logic của firmware, hoặc thêm các bước ghi dữ liệu mới.

## Yêu cầu trước

- Môi trường Python: kích hoạt `mamba activate rotary-inverted-pendulum` với các thư viện `numpy`, `scipy`, `pyserial`, `matplotlib`, `mujoco` đã được cài đặt.
- Firmware `LowLevelServer.ino` đã được nạp vào Arduino Nano. Thiết bị tự động phát hiện nam châm khi khởi động; firmware sẽ dừng ở hàm `setup()` cho đến khi nó phát hiện thấy nam châm của con lắc.
- Một khối cứng nhỏ (bìa cứng, xốp, gỗ) để kê dưới con lắc phục vụ việc thả quay tự do (free-swing release).
- Hình học của con lắc (khối lượng, COM, quán tính) được lấy từ `urdf/model.urdf` — không cần người dùng đo đạc trực tiếp. Nếu bạn đã lắp lại hoặc sửa đổi con lắc, hãy cập nhật thiết kế Onshape → xuất lại → cập nhật tệp URDF trước.

## Lộ trình khuyến nghị: toàn bộ quy trình (full pipeline)

```bash
cd RotaryInvertedPendulum-python/src/rl
python sysid_wizard.py
```

Các bước hướng dẫn lần lượt gồm:

1. **Hiệu chuẩn vị trí treo tĩnh (Tare hanging position).** Giữ con lắc treo thẳng đứng yên; chúng tôi lấy mẫu trong 3 giây và sử dụng giá trị trung vị làm điểm "treo vật lý = 0" trong hệ tọa độ đã lưu. Thao tác này xử lý độ trôi lệch của bộ tích lũy encoder trong firmware.
2. **Thả quay tự do (Free-swing) 3 lần.** Giữ chặt cánh tay động cơ, kê khối cứng dưới con lắc để nâng nó lên (ở một góc hợp lý bất kỳ — cảm biến sẽ ghi lại góc thả thực tế), sau đó rút nhanh khối cứng ra theo hướng vuông góc với mặt phẳng lắc. Thời gian ghi mặc định là 10 giây. Lặp lại 3 lần.
3. **Quét kiểm tra động cơ góc ±90° (Motor ±90° sanity sweep).** Kích hoạt động cơ và điều khiển nó quay theo chu trình `0 → +90° → 0 → −90° → 0` bằng các lệnh gia tốc. Quan sát thiết bị: cánh tay phải đạt vị trí xấp xỉ vuông góc ở các điểm cực đại. Nếu thấy cánh tay quay không tới, động cơ bước đang bị mất bước và bộ đếm vị trí của firmware không còn khớp với vị trí cơ khí thực tế — lỗi này sẽ chặn việc triển khai (hãy giảm `MOTOR_ACCELERATION` hoặc tăng biến trở Vref của DRV8825).
4. **Khớp mô hình + Vẽ đồ thị (Fit + plots).** Tổng hợp dữ liệu từ 3 bản ghi quay tự do, tính toán các tham số ma sát dựa trên hình học từ tệp URDF, ghi ra tệp `sysid_params.json`, tạo đồ thị `freeswing_compare.png` (đường mô phỏng đè lên bản ghi thực tế sử dụng các tham số vừa tính được) và đồ thị `motor_sweep.png` (vị trí mục tiêu của động cơ so với vị trí thực tế trong lượt quét kiểm tra).

Khối lượng con lắc, khoảng cách tâm khối (COM), và quán tính quanh tâm khối *không* còn là một phần của quy trình tính toán này nữa — chúng là các thuộc tính hình học của chi tiết và nằm trong tệp `urdf/model.urdf`, được phân tích bởi `pendulum_geometry.py`. Việc khớp mô hình quay tự do sẽ đối chiếu quán tính URDF với chu kỳ đo được (nếu chênh lệch lớn hơn 10% sẽ cảnh báo tệp URDF có thể đã cũ).

Toàn bộ quy trình mất khoảng ~10–15 phút. Đầu ra được lưu trong thư mục `sysid_runs/<timestamp>/` nằm cạnh tập lệnh wizard.

## Khớp lại mô hình từ bản ghi có sẵn (không cần thiết bị)

```bash
python sysid_wizard.py fit --in-dir sysid_runs/2026-05-20_090000
```

Sử dụng lệnh này khi tinh chỉnh hàm `sysid_core.derive_pendulum_friction` hoặc nghiên cứu kết quả khớp mô hình — thiết bị thực không cần sử dụng và người vận hành không cần có mặt.

## Chạy riêng lượt quét kiểm tra động cơ

```bash
python sysid_wizard.py validate-motor
```

Chỉ chạy lượt quét ±90° của động cơ và vẽ đồ thị. Hữu ích sau khi thay đổi cấu hình động cơ của firmware (giới hạn dòng điện, chế độ vi bước, giới hạn gia tốc) để xác nhận bộ đếm bước vẫn bám sát thực tế.

## Các tệp được tạo ra

```
sysid_runs/2026-05-20_HHMMSS/
├── metadata.json               # cổng kết nối, firmware, dấu thời gian, ...
├── tare.npz                    # bản ghi con lắc đứng yên → lấy điểm 0 treo tĩnh
├── free_run_1.npz              # các bản ghi quay tự do thô
├── free_run_2.npz
├── free_run_3.npz
├── motor_sweep.npz             # mục tiêu/thực tế động cơ trong lượt quét kiểm tra
├── freeswing_compare.png       # đồ thị đè sim-vs-real để xác thực
└── motor_sweep.png             # mục tiêu động cơ so với vị trí báo cáo của firmware
```

Tệp `sysid_params.json` (tệp duy nhất mà pipeline RL đọc) mặc định được ghi ở thư mục gốc của dự án.

## Cơ sở toán học xử lý

Hàm `fit_free_swing(t, θ)` (trong tệp `sysid_core.py`) tìm các điểm cực trị, tính toán chu kỳ và các hằng số suy giảm của dao động tắt dần. Nó cũng tính toán **chu kỳ biên độ nhỏ** T₀ thông qua hiệu chỉnh tích phân elip `T(θ_max) = T₀ · (2/π) · K(sin²(θ_max/2))` — việc này là bắt buộc vì các bản ghi được thực hiện ở biên độ hữu hạn (40–90°) trong khi công thức quán tính `I = m·g·d·T²/(4π²)` áp dụng cho biên độ nhỏ.

Hàm `derive_pendulum_friction(fit)` đọc hình học của con lắc (khối lượng, COM, I_com) từ `pendulum_geometry` (phân tích tệp `urdf/model.urdf`) và kết hợp với kết quả khớp mô hình để tính toán ma sát nhớt (viscous) và ma sát Coulomb. Nó cũng báo cáo cả quán tính dự đoán `inertia_predicted_kg_m2` (CAD: m·d² + I_com) và quán tính đo được `inertia_measured_kg_m2` (từ T₀: m·g·d/ω²), và hàm `validate_free_swing` sẽ cảnh báo nếu chúng chênh lệch quá 10% — dấu hiệu cho thấy tệp URDF đã cũ hoặc bản ghi bị nhiễu.

Ma sát nhớt sử dụng quán tính trục quay dự đoán từ CAD: b = 2·α·I_pred, trong đó α = b/(2I) là độ suy giảm đường bao đo được trực tiếp. Ma sát Coulomb được tính từ độ giảm Coulomb trên mỗi nửa chu kỳ đo được trong việc khớp đường bao.

## Khắc phục sự cố và Debug

- **Khớp mô hình quay tự do thất bại** (quá ít điểm cực trị, không có suy giảm, v.v.): Bản ghi bị hỏng — con lắc không chuyển động đủ nhiều, hoặc bạn đã thả tay gây rung lắc. Chỉ cần thử lại lượt chạy đó khi được nhắc.
- **Điểm 0 treo tĩnh bị trôi lệch giữa các bản ghi**: Bộ tích lũy của firmware là đơn điệu, điều này xảy ra nếu người vận hành xoay tay con lắc quá ±180° giữa các lượt chạy. Hãy reset firmware (rút cáp USB ra và cắm lại) rồi thu thập lại dữ liệu.
- **Khớp lại mô hình cho ra các số số khác nhau**: Điều này bình thường — các thay đổi nhỏ trong bộ khớp mô hình (fitter) có thể tích tụ. Đồ thị so sánh `freeswing_compare.png` là căn cứ thực tế cuối cùng: nếu đường bao thực tế và mô phỏng đè khít lên nhau, kết quả khớp là chính xác bất kể giá trị tuyệt đối thế nào.
