# CLAUDE.md

Tài liệu này cung cấp hướng dẫn cho Claude Code (claude.ai/code) khi làm việc với mã nguồn trong kho lưu trữ này.

## Các sáng kiến đang hoạt động (Active initiatives)

- **Bộ điều khiển RL**: nỗ lực nhiều giai đoạn để thay thế PID tinh chỉnh thủ công bằng chính sách swing-up + cân bằng (balance) tự học. Kế hoạch hiện tại, trạng thái giai đoạn và nhật ký quyết định nằm trong tệp `RL_PLAN.md` ở thư mục gốc của repo. Hãy đọc tệp đó trước khi làm việc với bất kỳ thứ gì dưới thư mục `RotaryInvertedPendulum-arduino/LowLevelServer/`, `RotaryInvertedPendulum-arduino/RLControl/`, hoặc `RotaryInvertedPendulum-python/src/rl/`. Các tài liệu đồng hành:
  - `docs/rl_transitions.md` — đặc tả chuyển trạng thái `(s, a, r, s')` bằng ngôn ngữ tự nhiên.
  - `docs/async_control_architecture.md` — runtime đa luồng giữ tần số điều khiển (control rate) nghiêm ngặt trong quá trình tinh chỉnh (fine-tuning).
  - `docs/control_rate_selection.md` — cách chọn `control_freq_hz` và `max_action_delta_rad` từ các phép đo sysid.
  - `docs/sysid_runbook.md` — quy trình đo lường cho các đầu vào mà hai tài liệu trên phụ thuộc vào.

## Nơi lưu trữ các tham số vật lý (Physical parameters)

Hệ thống con lắc ngược quay có một nguồn chân lý duy nhất (source of truth) cho mỗi nhóm tham số. Cập nhật một số ở bất kỳ nơi nào khác đều là lỗi — chuỗi truyền dẫn là:

- **Hình học của con lắc** (khối lượng - mass, tâm khối - COM, tensor quán tính - inertia tensor): được thiết kế trong Onshape → xuất sang `urdf/model.urdf` → được phân tích lúc import bởi `RotaryInvertedPendulum-python/src/rl/pendulum_geometry.py` → được sử dụng bởi `pendulum_env.py` (mô phỏng MuJoCo + DR), `sysid_core.py` (tính toán ma sát, kiểm tra chéo với chu kỳ đo được), và stack Julia (trực quan hóa MeshCat, RigidBodyDynamics MPC). Để thay đổi khối lượng/COM/quán tính của con lắc, hãy chỉnh sửa trong Onshape → xuất tệp URDF; không sửa gì khác.

- **Trạng thái động lực học của từng rig** (ma sát nhớt + Coulomb): được đo bởi `sysid_wizard.py` từ bản ghi tự quay tự do trên phần cứng thực tế, ghi vào `RotaryInvertedPendulum-python/src/rl/sysid_params.json`, tải vào `PendulumParams` cùng với các hằng số URDF. Các giá trị này thay đổi giữa các lần lắp ráp lại (ổ đỡ, mỡ bôi trơn, nhiệt độ) và là các đại lượng duy nhất mà pipeline sysid đo lường.

- **Hình học của cánh tay (arm)** (chiều dài, khối lượng, COM): hiện là các hằng số được hard-code trong `pendulum_env.py` (`ARM_*`). Khi được xác thực bằng CAD, sẽ tuân theo mẫu của con lắc — đọc từ liên kết `arm` trong `urdf/model.urdf`.

- **Hằng số phần cứng/firmware** (gia tốc tối đa của motor, độ phân giải AS5600, giới hạn dừng vật lý): các hằng số module trong `pendulum_env.py`, với firmware Arduino là nguồn cấp trên (upstream) cho các giá trị phía động cơ.

## Tổng quan

Đây là dự án điều khiển con lắc ngược quay (rotary inverted pendulum) - một bài toán lý thuyết điều khiển cổ điển chứng minh khả năng ổn định hệ thống. Hệ thống gồm một con lắc gắn trên một đế quay, con lắc phải được giữ cân bằng thẳng đứng bằng cách điều khiển chuyển động quay của đế. Dự án bao gồm các thiết kế cơ khí (có thể in 3D), linh kiện điện tử (dựa trên Arduino) và nhiều giải thuật điều khiển khác nhau.

## Cấu trúc dự án

Kho lưu trữ được tổ chức thành ba thành phần phần mềm chính:

- **RotaryInvertedPendulum-arduino/**: Mã nguồn C++ Arduino cho vi điều khiển (Arduino Nano)
  - `LowLevelServer/`: Server cấp thấp cho hoạt động điều khiển từ máy tính thông qua kết nối serial
  - `PIDControl/`: Bộ điều khiển PID độc lập chạy trực tiếp trên Arduino
  - `TestEncoder/`, `TestMotor/`, `TestSerial/`, v.v.: Các bản sketch kiểm tra phần cứng

- **RotaryInvertedPendulum-julia/**: Các thuật toán điều khiển bằng Julia và trực quan hóa
  - `src/`: Mã nguồn gói Julia chính
  - `notebooks/`: Các Jupyter notebook để thử nghiệm (ví dụ: phát triển MPC)

- **RotaryInvertedPendulum-python/**: Các triển khai điều khiển bằng Python
  - `src/`: Mã nguồn điều khiển Python (điều khiển bằng gamepad)
  - `test/`: Các tập lệnh kiểm tra (test scripts)

Các thư mục bổ sung:
- `meshes/`: Các tệp STL in 3D cho linh kiện cơ khí
- `diagrams/`: Sơ đồ mạch và sơ đồ hệ thống
- `urdf/`: Các tệp mô tả mô hình robot

## Kiến trúc phần cứng

Hệ thống sử dụng:
- **Arduino Nano**: Vi điều khiển để đọc cảm biến và điều khiển động cơ
- **Cảm biến mã hóa vòng quay từ tính AS5600 (AS5600 Magnetic Encoder)**: Đo góc con lắc (giao tiếp I2C)
- **Động cơ bước (NEMA17 Stepper Motor)**: Quay cánh tay đế (thông qua driver như DRV8825/A4988/TMC2209)
- **Thư viện AccelStepper**: Điều khiển động cơ bước với các đặc tính gia tốc

Giao tiếp giữa Arduino và máy tính qua cổng serial với tốc độ baud 2,000,000.

## Các phương pháp điều khiển

Hai kiến trúc điều khiển chính:

1. **Điều khiển trên thiết bị** (Arduino): Bộ điều khiển PID chạy hoàn toàn trên Arduino Nano
   - Nhỏ gọn, không cần máy tính
   - Khả năng tính toán hạn chế
   - Xem: `RotaryInvertedPendulum-arduino/PIDControl/PIDControl.ino`

2. **Điều khiển trên máy tính** (Julia/Python): Arduino đóng vai trò là server cấp thấp (low-level server)
   - Khả năng tính toán cao cho các thuật toán nâng cao (MPC, LQR)
   - Yêu cầu kết nối USB với máy tính
   - Mã nguồn Arduino: `LowLevelServer/LowLevelServer.ino`
   - Mã nguồn client: Các tệp Julia trong `src/` hoặc Python trong `RotaryInvertedPendulum-python/`

## Giao thức truyền thông Serial

### Giao thức dạng văn bản (PIDControl)
Các lệnh được gửi dưới dạng chuỗi văn bản:
- `"1"`: Kiểm tra xem đã sẵn sàng chưa
- `"2"`: Lấy vị trí động cơ
- `"3"`: Lấy vị trí con lắc
- `"4 <position>"`: Đặt vị trí động cơ mục tiêu
- `"5"`: Khởi động động cơ
- `"6"`: Dừng động cơ

### Giao thức dạng nhị phân (LowLevelServer)
Các lệnh là các byte đơn lẻ:
- `0x01`: Kiểm tra sẵn sàng
- `0x02`: Lấy trạng thái (trả về thời gian, vị trí động cơ, vị trí con lắc dưới dạng số thực float)
- `0x03`: Đặt mục tiêu (yêu cầu số thực float 4-byte tính bằng radian)
- `0x04`: Kích hoạt động cơ (engage)
- `0x05`: Ngắt kích hoạt động cơ (disengage)

## Phát triển với Julia

### Thiết lập
```bash
cd RotaryInvertedPendulum-julia
julia --project=.
```

Trong Julia REPL:
```julia
using Pkg
Pkg.instantiate()  # Cài đặt các thư viện phụ thuộc
```

### Chạy các Script điều khiển

Điều khiển PID từ Julia:
```julia
using RotaryInvertedPendulum
pid_control()  # Mặc định: baud 2000000, tần số điều khiển 200 Hz
```

Client server cấp thấp có trực quan hóa:
```bash
julia --project=. ../RotaryInvertedPendulum-arduino/LowLevelServer/client.jl --visualise
```

### Các tệp Julia quan trọng

- `RotaryInvertedPendulum.jl`: Module chính, định nghĩa các lệnh serial và giao tiếp Arduino
- `control_pid.jl`: Triển khai bộ điều khiển PID giao tiếp qua cổng serial
- `control_gamepad.jl`: Điều khiển thủ công bằng Gamepad
- `mpc.jl`: Triển khai bộ điều khiển dự báo mô hình (Model Predictive Control) với tuyến tính hóa hệ thống
- `utils.jl`: Các hàm tiện ích
- `precompile.jl`: Biên dịch trước gói để khởi động nhanh hơn

### Các thư viện phụ thuộc
Gói Julia sử dụng:
- `LibSerialPort`: Giao tiếp serial với Arduino
- `RigidBodyDynamics`: Để mô hình hóa động lực học hệ thống
- `ForwardDiff`: Đạo hàm tự động (automatic differentiation) cho tuyến tính hóa MPC
- `MeshCat`, `MeshCatMechanisms`: Trực quan hóa 3D
- `Joysticks`: Hỗ trợ Gamepad
- `Plots`: Vẽ đồ thị dữ liệu

## Phát triển với Arduino

### Yêu cầu trước
Các thư viện cần thiết (cài đặt qua Arduino IDE Library Manager):
- [AccelStepper](https://www.airspayce.com/mikem/arduino/AccelStepper/)
- [AS5600](https://github.com/Seeed-Studio/Seeed_Arduino_AS5600) (được bao gồm trong thư mục `libs/`)

### Nạp chương trình cho Arduino
1. Mở tệp `.ino` trong Arduino IDE
2. Chọn Board: "Arduino Nano"
3. Chọn Port: `/dev/cu.usbserial-*` (macOS) hoặc cổng COM phù hợp (Windows)
4. Tải lên (Upload) sketch

### Các khái niệm Arduino quan trọng

**Cấu hình động cơ bước (Stepper Motor):**
- Microstepping: 8 (mặc định) → 1600 bước/vòng quay (steps/revolution)
- Chân Enable bị đảo ngược (DRV8825 sử dụng chân enable tích cực mức thấp - active-low)
- Tốc độ tối đa: 200,000 steps/sec
- Gia tốc: 100,000 steps/sec²

**Cảm biến AS5600 Encoder:**
- Cung cấp độ phân giải 12-bit (giá trị thô 0-4095)
- Ánh xạ tới góc 0-360° hoặc 0-2π radian
- Xử lý việc theo dõi nhiều vòng quay với logic wraparound
- Kiểm tra cường độ nam châm khi khởi động

**Tham số điều khiển PID:**
- Tần số điều khiển: 200-1000 Hz (tùy thuộc vào cách triển khai)
- Các hệ số Kp, Ki, Kd được tinh chỉnh thủ công trong `PIDControl.ino`: Kp=2.2, Ki=1.6, Kd=0.005
- Giới hạn vị trí động cơ: ±90° từ vị trí bắt đầu (tránh làm xoắn và đứt dây)
- Biên kích hoạt điều khiển (engagement margin): ±25° từ phương thẳng đứng

## Phát triển với Python

Phiên bản Python ít được phát triển hơn nhưng bao gồm điều khiển bằng gamepad và các kịch bản kiểm tra serial. Cài đặt các thư viện phụ thuộc và chạy các kịch bản từ thư mục `RotaryInvertedPendulum-python/`.

## Các tác vụ phát triển thường gặp

### Kiểm tra phần cứng
- **Kiểm tra encoder**: Nạp chương trình `TestEncoder/TestEncoder.ino`, mở Serial Monitor/Plotter
- **Kiểm tra động cơ**: Nạp chương trình `TestMotor/TestMotor.ino`
- **Kiểm tra giao tiếp serial**: Nạp chương trình `TestSerial/TestSerial.ino`

### Kiểm tra phần cứng trong vòng lặp (Hardware-in-the-Loop Testing)
Để kiểm tra tự động khi đã kết nối phần cứng, hãy sử dụng kịch bản giám sát serial:

```bash
./RotaryInvertedPendulum-arduino/scripts/monitor_serial.sh <port> <baud_rate> <duration>
```

Tập lệnh này xử lý đúng cách việc khởi động lại Arduino khi kết nối serial và xóa dữ liệu đệm cũ để cung cấp đầu ra sạch. Hữu ích cho việc xác minh hành vi của Arduino trong quá trình phát triển mà không cần can thiệp thủ công.

Ví dụ:
```bash
./RotaryInvertedPendulum-arduino/scripts/monitor_serial.sh /dev/cu.usbserial-10 115200 10
```

**Lưu ý:** Cách tiếp cận này tránh được các vấn đề phổ biến khi sử dụng trực tiếp lệnh `cat` hoặc `stty` vốn có thể gây ra hiện tượng reset kép hoặc lấy dữ liệu đệm cũ.

### Các vấn đề về cổng Serial
Trên macOS, Arduino thường xuất hiện dưới dạng `/dev/cu.usbserial-110` hoặc tương tự. Cập nhật chuỗi cổng trong mã Julia/Python nếu có sự khác biệt.

### Giới hạn dòng điện (Current Limiting)
Đặt giới hạn dòng điện của driver động cơ bước thành 0.9A (90% công suất định mức 1A của động cơ) bằng biến trở tinh chỉnh (potentiometer) trên bo mạch. Công thức tính Vref khác nhau tùy theo driver - xem README.md.

## Ghi chú về lý thuyết điều khiển

Hệ thống triển khai phương pháp không gian trạng thái (state-space) trong đó:
- Trạng thái: `[motor_angle, pendulum_angle, motor_velocity, pendulum_velocity]`
- Đầu vào điều khiển: mô-men xoắn động cơ (được chuyển đổi thành các lệnh vị trí cho động cơ bước)

**Triển khai MPC** (`mpc.jl`):
- Tuyến tính hóa động lực học phi tuyến bằng cách sử dụng `RigidBodyDynamics` và `ForwardDiff`
- Sử dụng tích phân RK4 cho động lực học thời gian rời rạc
- Điểm tuyến tính hóa: con lắc thẳng đứng (π radian), động cơ ở vị trí gốc (origin)

**Giản đồ trạng thái (State Machine)** (cả Arduino và Julia PID):
- `WAITING`: Động cơ bị vô hiệu hóa, chờ con lắc gần phương thẳng đứng
- `BALANCING`: Động cơ hoạt động, tích cực điều khiển

## URDF và trực quan hóa

Thư mục `urdf/` chứa các tệp mô tả robot được sử dụng bởi `RigidBodyDynamics.jl` để tính toán động lực học và `MeshCat` để trực quan hóa 3D.

## Cấu hình cổng Serial

Tốc độ baud Arduino: 2,000,000 (tốc độ cao để điều khiển thời gian thực)
- Thời gian chờ đọc (Read timeout): 50ms (thông thường)
- Thời gian chờ ghi (Write timeout): 10ms (thông thường)

Luôn gọi `wait_until_ready(arduino)` sau khi mở kết nối serial để đồng bộ hóa với Arduino.
