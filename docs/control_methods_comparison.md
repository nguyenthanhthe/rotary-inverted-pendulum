# So Sánh Các Bộ Điều Khiển Cho Con Lắc Ngược Quay

Tài liệu này cung cấp bảng so sánh chi tiết các phương pháp điều khiển con lắc ngược quay (Furuta Pendulum), phân tích ưu và nhược điểm của từng phương pháp và chỉ ra bộ điều khiển tối ưu cho từng mục đích sử dụng.

---

## 1. Các Bộ Điều Khiển Thay Thế Phổ Biến

Ngoài Học tăng cường (RL), con lắc ngược quay có thể được điều khiển bằng các phương pháp điều khiển lý thuyết cổ điển và hiện đại:

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

## 2. Bảng So Sánh Hiệu Quả Điều Khiển

| Tiêu chí | Học Tăng Cường (RL - SAC) | LQR + Energy Control | MPC (Dự báo mô hình) | Cascaded PID |
| :--- | :--- | :--- | :--- | :--- |
| **Tự động Swing-up** | **Rất tốt** (Học được chính sách hợp nhất cả swing-up và balance) | **Tốt** (Cần lập trình logic chuyển giao trạng thái thủ công) | **Rất tốt** (Nếu giải phi tuyến trực tuyến) | **Kém** (Rất khó tự swing-up mịn màng) |
| **Khả năng chống nhiễu**| **Rất cao** (Nhờ Xáo trộn miền - Domain Randomization) | **Trung bình** (Nhạy cảm với sai số mô hình vật lý) | **Cực cao** (Nhờ tối ưu hóa liên tục từng bước) | **Thấp** (Dễ bị dao động cộng hưởng nếu bị gõ mạnh) |
| **Yêu cầu chip xử lý** | **Thấp** (Sau khi chưng cất, chỉ chạy 1 mạng MLP nhỏ 5KB trên Nano) | **Cực thấp** (Chỉ là phép nhân ma trận đơn giản trên Arduino Nano) | **Rất cao** (Không thể chạy trực tiếp trên vi điều khiển 8-bit như Nano) | **Cực thấp** (Chỉ có vài phép cộng nhân cơ bản) |
| **Yêu cầu mô hình toán**| **Trung bình** (Chỉ cần mô phỏng MuJoCo tương đối chính xác) | **Rất cao** (Cần phương trình Euler-Lagrange chính xác) | **Cực kỳ cao** (Cần mô hình động lực học thời gian thực chính xác) | **Không yêu cầu** (Tinh chỉnh thử - sai trực tiếp trên phần cứng) |
| **Độ khó thiết kế** | **Trung bình** (Tốn thời gian huấn luyện và cấu hình tham số DR) | **Cao** (Đòi hỏi kiến thức toán lý thuyết điều khiển hiện đại) | **Cực kỳ cao** (Đòi hỏi thuật toán tối ưu hóa số trực tuyến) | **Thấp** (Dễ làm nhưng tốn thời gian xoay biến trở/thử sai) |

### Bộ điều khiển nào tốt nhất?
*   **Để cân bằng tĩnh tối ưu**: **LQR** là tốt nhất vì nó tuyến tính, tối ưu hóa năng lượng cực kỳ mịn màng và động cơ gần như đứng yên khi cân bằng (Calm Attractor).
*   **Để tích hợp toàn diện (End-to-End)**: **RL (SAC)** là tốt nhất vì nó tự động tìm ra đường đi tối ưu để swing-up và tự chuyển sang thăng bằng mượt mà mà không cần lập trình các ngưỡng kích hoạt chuyển giao (state transition thresholds) phức tạp bằng tay.
