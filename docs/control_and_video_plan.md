# So Sánh Bộ Điều Khiển & Kế Hoạch Video Giới Thiệu (Google-style)

Tài liệu này cung cấp bảng so sánh chi tiết các phương pháp điều khiển con lắc ngược quay (Furuta Pendulum) và xây dựng kịch bản chi tiết cho video giới thiệu 1 phút theo phong cách tối giản, hiện đại của Google.

---

## I. So Sánh Các Bộ Điều Khiển Cho Con Lắc Ngược Quay

Ngoài Học tăng cường (RL), con lắc ngược quay có thể được điều khiển bằng các phương pháp điều khiển lý thuyết cổ điển và hiện đại:

### 1. Các bộ điều khiển thay thế phổ biến
*   **LQR (Linear Quadratic Regulator)**: 
    *   *Nguyên lý*: Tuyến tính hóa hệ thống phi tuyến quanh điểm cân bằng thẳng đứng ($0,0,0,0$), tìm ma trận phản hồi trạng thái $K$ để tối ưu hóa hàm chi phí toàn phương của trạng thái và tín hiệu điều khiển.
    *   *Đặc trưng*: Cực kỳ ổn định khi cân bằng, nhưng **không thể tự swing-up** (chỉ giữ thăng bằng khi đã được đưa sát điểm thẳng đứng).
*   **Energy-based Control (Điều khiển dựa trên Năng lượng)**:
    *   *Nguyên lý*: Tính toán năng lượng hiện tại của con lắc và tác động lực để bơm thêm hoặc triệt tiêu năng lượng, đưa con lắc từ trạng thái treo thẳng đứng lên điểm chết trên.
    *   *Đặc trưng*: Chuyên dùng để **swing-up**, nhưng không thể giữ thăng bằng (phải chuyển giao điều khiển sang LQR/PID khi lên tới đỉnh).
*   **MPC (Model Predictive Control)**:
    *   *Nguyên lý*: Giải bài toán tối ưu hóa động lượng trực tuyến ở mỗi bước thời gian dựa trên mô hình dự báo của hệ thống.
    *   *Đặc trưng*: Xử lý rất tốt các giới hạn vật lý (như góc giới hạn của cánh tay, tốc độ tối đa), cực kỳ mạnh mẽ nhưng đòi hỏi chip xử lý cực mạnh.
*   **PID song song/phân tầng (Cascaded PID)**:
    *   *Nguyên lý*: Dùng 2 vòng PID độc lập: vòng ngoài ổn định góc con lắc, vòng trong ổn định vị trí cánh tay motor.
    *   *Đặc trưng*: Đơn giản, không cần mô hình toán học, nhưng rất khó tinh chỉnh các tham số $K_p, K_i, K_d$ cho hệ MIMO (nhiều đầu vào, nhiều đầu ra) và dễ mất ổn định khi bị nhiễu lớn.

---

### 2. Bảng So Sánh Hiệu Quả Điều Khiển

| Tiêu chí | Học Tăng Cường (RL - SAC) | LQR + Energy Control | MPC (Dự báo mô hình) | Cascaded PID |
| :--- | :--- | :--- | :--- | :--- |
| **Tự động Swing-up** | **Rất tốt** (Học được chính sách hợp nhất cả swing-up và balance) | **Tốt** (Cần lập trình logic chuyển giao trạng thái thủ công) | **Rất tốt** (Nếu giải phi tuyến trực tuyến) | **Kém** (Rất khó tự swing-up mịn màng) |
| **Khả năng chống nhiễu**| **Rất cao** (Nhờ Xáo trộn miền - Domain Randomization) | **Trung bình** (Nhạy cảm với sai số mô hình vật lý) | **Cực cao** (Nhờ tối ưu hóa liên tục từng bước) | **Thấp** (Dễ bị dao động cộng hưởng nếu bị gõ mạnh) |
| **Yêu cầu chip xử lý** | **Thấp** (Sau khi chưng cất, chỉ chạy 1 mạng MLP nhỏ 5KB trên Nano) | **Cực thấp** (Chỉ là phép nhân ma trận đơn giản trên Arduino Nano) | **Rất cao** (Không thể chạy trực tiếp trên vi điều khiển 8-bit như Nano) | **Cực thấp** (Chỉ có vài phép cộng nhân cơ bản) |
| **Yêu cầu mô hình toán**| **Trung bình** (Chỉ cần mô phỏng MuJoCo tương đối chính xác) | **Rất cao** (Cần phương trình Euler-Lagrange chính xác) | **Cực kỳ cao** (Cần mô hình động lực học thời gian thực chính xác) | **Không yêu cầu** (Tinh chỉnh thử - sai trực tiếp trên phần cứng) |
| **Độ khó thiết kế** | **Trung bình** (Tốn thời gian huấn luyện và cấu hình tham số DR) | **Cao** (Đòi hỏi kiến thức toán lý thuyết điều khiển hiện đại) | **Cực kỳ cao** (Đòi hỏi thuật toán tối ưu hóa số trực tuyến) | **Thấp** (Dễ làm nhưng tốn thời gian xoay biến trở/thử sai) |

### Bộ điều khiển nào tốt nhất?
*   **Để cân bằng tĩnh tối ưu**: **LQR** là tốt nhất vì nó tuyến tính, tối ưu hóa năng lượng cực kỳ mịn màng và động cơ gần như đứng yên khi cân bằng ( Calm Attractor).
*   **Để tích hợp toàn diện (End-to-End)**: **RL (SAC)** là tốt nhất vì nó tự động tìm ra đường đi tối ưu để swing-up và tự chuyển sang thăng bằng mượt mà mà không cần lập trình các ngưỡng kích hoạt chuyển giao (state transition thresholds) phức tạp bằng tay.

---

## II. Kịch Bản Video Giới Thiệu 1 Phút (Phong Cách Google)

*   **Thời lượng**: 60 giây.
*   **Phong cách**: Tối giản, hiện đại, nhịp điệu nhanh (Fast-paced), tập trung vào chuyển động cơ khí mượt mà, đồ họa số liệu hiển thị trực quan (neon/minimalist HUD overlay) và nhạc điện tử lôi cuốn (uplifting/ambient electronic).

### Bảng Phân Cảnh (Storyboard & Script)

| Thời gian | Hình ảnh (Visual) | Đồ họa chồng (Overlay) | Âm thanh (Audio) | Ý nghĩa truyền tải |
| :--- | :--- | :--- | :--- | :--- |
| **00:00 - 00:08** | Cận cảnh (Close-up) con lắc đang đứng yên treo thẳng đứng. Bất ngờ cánh tay motor nhích nhẹ, đánh đu 2 nhịp và dựng đứng con lắc thăng bằng hoàn hảo. | Chữ tối giản: **"An unstable system."** dịch chuyển thành **"Solved by AI."** | Tiếng bass thả trầm (deep drop), nhạc nền bắt đầu nổi lên nhẹ nhàng. | **Gây ấn tượng đầu tiên (Hook):** Phô diễn trực tiếp tính năng khó nhất của con lắc. |
| **00:08 - 00:18** | Chuyển cảnh nhanh (Fast cuts): Thiết kế 3D xoay trên máy tính (OnShape), máy in 3D đang phun nhựa PLA tạo phần cánh tay và chân đế nhựa. | Các đường lưới CAD 3D phát sáng neon chạy dọc cơ cấu con lắc. | Nhạc tăng nhịp độ (build-up), tiếng máy in 3D cách điệu nghệ thuật. | **Thiết kế cơ khí:** Cho thấy đây là dự án DIY mở, có thể chế tạo tại nhà. |
| **00:18 - 00:28** | Cận cảnh các linh kiện điện tử nổi bật: Mạch Arduino Nano nhỏ gọn, Driver DRV8825, và cảm biến từ trường AS5600 ở khớp xoay con lắc. | Chữ chú thích các thành phần: `Arduino Nano`, `AS5600 12-bit Encoder`, `NEMA 17 Stepper`. | Tiếng beep điện tử nhẹ nhàng theo nhịp nhạc. | **Thiết kế điện tử:** Tối giản phần cứng với chi phí rẻ dưới $20. |
| **00:28 - 00:43** | Màn hình chia đôi (Split screen): Bên trái là môi trường mô phỏng vật lý MuJoCo (với các thanh tham số xáo trộn miền nhảy liên tục); bên phải là robot thực đang lắc lư bám theo. | Các chỉ số thực tế: `Domain Randomization (DR)`, `SAC Policy (5->32->32->1)`. | Nhạc đạt đến cao trào (căng tràn năng lượng). | **Reinforcement Learning:** Giải thích cách chuyển giao từ mô phỏng sang thực tế (Sim-to-Real). |
| **00:43 - 00:53** | Bàn tay rút sợi cáp USB nối với máy tính ra. Con lắc vẫn thăng bằng độc lập. Ngón tay gõ nhẹ vào thanh lắc, con lắc nghiêng đi rồi lập tức tự điều chỉnh về vị trí đứng thẳng. | Icon viên pin hoặc biểu tượng **"Standalone (Tether-free)"**, đồ họa radar sóng sửa sai (Error Correction). | Nhạc nền nhẹ dần, tiếng động cơ bước ro ro mượt mà. | **Demo thực tế:** Minh chứng độ ổn định và tính độc lập của mô hình nén chạy trên vi điều khiển yếu. |
| **00:53 - 01:00** | Logo dự án "Rotary Inverted Pendulum DIY" hiện lên tối giản trên nền tối. Kèm đường dẫn Github repo. | **"Build. Train. Balance."**<br/>`github.com/nguyenthanhthe/rotary-inverted-pendulum` | Âm thanh kết thúc ngân vang (ringing synth tail) kiểu Google. | **Kêu gọi hành động (Call to action):** Truy cập mã nguồn mở để cùng làm. |
