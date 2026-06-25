# Ghi chú in 3D

Các ghi chú thực tế để in các bộ phận cơ khí. Các tệp STL nằm trong thư mục [`../meshes/`](../meshes); nguồn thiết kế OnShape được liên kết từ [`BOM.md`](BOM.md).

## Cài đặt (Settings)

Sử dụng nhựa PLA hoặc PETG, đầu in (nozzle) 0.4 mm, độ dày lớp (layer) 0.2 mm (sử dụng cấu hình mặc định của Bambu Studio). Sử dụng support dạng cây ('Tree' supports) cho phần vỏ hộp (đế - base, nắp - lid) và cánh tay quay (arm).

## Bộ phận con lắc (Pendulum link) — Tạm dừng in để chèn đồng xu

Bộ phận liên kết con lắc (pendulum link) có một khe được thiết kế vừa với đồng xu 2-pence của Anh (nặng 7.12 g) để làm khối lượng đầu mút (tip mass). Hãy cấu hình phần mềm cắt lớp (slicer) của bạn để **tạm dừng ở lớp 21 (pause on layer 21)** — thả đồng xu vào khe, sau đó tiếp tục in. Các lớp in tiếp theo sẽ bịt kín đồng xu bên trong.

Nếu bạn bỏ qua bước tạm dừng này, bộ phận liên kết sẽ được in với một khoảng trống bên trong nơi lẽ ra phải có đồng xu: khi đó khối lượng và quán tính của nó sẽ không khớp với những gì được khai báo trong [`urdf/model.urdf`](../urdf/model.urdf), và chính sách điều khiển (policy) sẽ không thể chuyển giao thành công từ mô phỏng sang thực tế (sim-to-real).

## Lưu ý khi hàn

Hãy cắt dây nối dài hơn một chút so với mức bạn nghĩ là cần thiết. Phần dây thừa rất dễ quản lý và thu gọn; ngược lại, một sợi dây thiếu chỉ vài milimet sẽ biến mối hàn tiếp theo thành một cuộc vật lộn khó khăn.

## Gợi ý phối màu in 3D & Thẩm mỹ (Color & Aesthetics Recommendations)

Để đạt được vẻ ngoài cao cấp, chuyên nghiệp (tương tự như các thiết bị dùng trong phòng thí nghiệm) và phù hợp với các decal tùy chỉnh như logo UET-VNU và thang đo góc trên nắp, chúng tôi khuyên bạn nên sử dụng các phối màu sau:

### Lựa chọn 1: "UET High-Tech Lab" (Tối giản & Hiện đại)
*   **Đế (Base):** **Trắng mờ (Matte White)** hoặc **Xám sáng (Light Grey)**. Lựa chọn này tạo ra một nền sạch sẽ làm nổi bật nhãn dán logo UET-VNU màu xanh dương và trắng.
*   **Nắp (Lid):** **Đen nhám (Matte Black)**. Nhãn dán thang đo góc màu đen (chữ trắng trên nền đen) sẽ hòa hợp hoàn toàn với nhựa đen nhám, giúp che đi các mép dán của nhãn.
*   **Cánh tay (Arm):** **Xanh hoàng gia / Xanh kim loại (Royal Blue / Metallic Blue - màu xanh UET)**. Phù hợp với màu thương hiệu của trường đại học và làm nổi bật phần cánh tay quay.
*   **Con lắc (Pendulum):** **Đen nhám (Matte Black)** hoặc **Bạc (Silver)**. Tạo sự tương phản tốt với cánh tay màu xanh để có thể nhìn rõ hành động lắc.

### Lựa chọn 2: "Academic Performance" (Đậm tính cơ khí - Tương tự ảnh mẫu)
*   **Đế (Base):** **Xanh hải quân (Navy Blue)** hoặc **Đen nhám (Matte Black)**. Phù hợp nếu decal UET-VNU của bạn có viền/nền màu trắng.
*   **Nắp (Lid):** **Đen nhám (Matte Black)** để khớp hoàn hảo với nền nhãn dán thang đo góc màu đen.
*   **Cánh tay (Arm):** **Đỏ kim loại (Metallic Red)** hoặc **Cam Neon (Vibrant Orange)**. Màu sắc có độ tương phản cao (như trong ảnh mẫu) giúp hỗ trợ theo dõi bằng camera và quan sát bằng mắt thường.
*   **Con lắc (Pendulum):** **Trắng mờ (Matte White)** hoặc **Bạc (Silver)**.

---

## Cân Chỉnh Kích Thước & Kinh Nghiệm Thực Tế (Critical Mechanical Adjustments)

Để hệ thống hoạt động chính xác và không có sai số cơ học, bạn nên áp dụng các điều chỉnh thực tế sau khi in 3D:

### 1. Khử độ rơ cơ khí đầu trục chữ D (D-hole backlash prevention)
*   **Vấn đề**: Trong bản vẽ thiết kế CAD gốc, lỗ chữ D trên cánh tay (Arm) được thiết kế với đường kính 5.3 mm và khoảng cách cạnh phẳng là 4.8 mm. Khi in 3D thực tế, dung sai của máy in sẽ làm lỗ này hơi rộng hơn một chút, dẫn đến **độ rơ cơ khí (backlash)** giữa trục động cơ bước và cánh tay. Độ rơ này sẽ trực tiếp làm giảm chất lượng điều khiển LQR hoặc RL (khi motor đảo chiều, cánh tay không di chuyển ngay lập tức).
*   **Giải pháp**: Hãy chỉnh sửa bản vẽ CAD (ví dụ trên OnShape) hoặc bù trừ dung sai lúc in để giảm kích thước lỗ chữ D xuống **đường kính 5.1 mm** và **khoảng cách cạnh phẳng 4.6 mm** (thông số thực tế của trục motor bước là 5.0 mm đường kính và 4.5 mm phẳng). Điều này giúp cánh tay lắp khít hoàn toàn vào trục và loại bỏ hoàn toàn độ rơ.

### 2. Hướng in Nắp (Lid) để có bề mặt đẹp nhất
*   **Vấn đề**: Việc in nắp theo chiều thuận (mặt trên hướng lên) sẽ yêu cầu rất nhiều support bên dưới và làm bề mặt nắp bị sần sùi do tiếp xúc với lớp support, làm mất đi độ thẩm mỹ cao cấp của thiết bị.
*   **Giải pháp**: Hãy loại bỏ các chi tiết chặn/luồn dây (Stop/Wire-guide) trong file CAD (sau lệnh Extrude 4) và **in úp ngược nắp** (mặt trên của nắp áp thẳng xuống bàn in). Điều này giúp mặt trên của nắp phẳng mịn tuyệt đối và hoàn toàn không cần dùng vật liệu support. Các chi tiết Stop/Wire-guide nếu cần thiết có thể thiết kế và in riêng rồi ghép vào sau (thực tế vận hành thường không cần).

### 3. Đi dây cảm biến tránh vướng víu
*   **Giải pháp**: Thay vì luồn dây qua bộ dẫn hướng dây (Wire-guide) trên nắp, bạn hãy lắp cánh tay nhô cao lên khoảng 8 mm so với bề mặt nắp, sau đó quấn dây của cảm biến góc (AS5600) xung quanh trục động cơ vài vòng trước khi đi dây ra ngoài. Cách này cho phép cánh tay xoay tự do nhiều vòng mà không bị căng dây hay cản trở hành trình cơ học.

