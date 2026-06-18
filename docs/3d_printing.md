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
