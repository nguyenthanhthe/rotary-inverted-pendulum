# Danh sách linh kiện (Bill of Materials - BOM)

Các linh kiện cần thiết để chế tạo một bộ con lắc ngược quay. Cập nhật lần cuối ngày 18/06/2026 dựa trên linh kiện mua ở sàn thương mại điện tử Shopee Việt Nam và cửa hàng linh kiện lkcg.vn.

**Ước tính chi phí**  
- Tổng chi phí mua bổ sung linh kiện (chưa có sẵn): **khoảng 350.000đ** (đã bao gồm dây điện Teflon 24AWG mới mua).
- Tổng chi phí chế tạo mới toàn bộ từ đầu (bao gồm cả các linh kiện đã có sẵn trong danh sách của bạn): **khoảng 490.000đ** — rẻ hơn khoảng 300 lần so với bộ thiết bị Quanser QUBE 2 thương mại (giá niêm yết khoảng 140.000.000đ).

**Cơ sở thiết kế**  
Để hiểu rõ *tại sao* mỗi linh kiện điện tử được chọn (tính toán nguồn điện, chọn động cơ, so sánh các driver, tụ lọc nguồn, v.v.), hãy xem tài liệu [`electronics_design.md`](electronics_design.md). Bản BOM này thuần túy là tài liệu tham khảo để mua sắm.

**Nhà cung cấp**  
Các linh kiện dưới đây được liên kết trực tiếp tới các cửa hàng trên **Shopee Việt Nam** và trang web **lkcg.vn**.

---

## Linh kiện điện tử (Electronics)

| Linh kiện | Thông số (Spec) | Giá (đ) | Nguồn | Ghi chú |
| ---------------------- | ----------------------------------------------- | --------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vi điều khiển | Arduino Nano (ATmega328P, 16 MHz, CH340, USB-C) | Đã có sẵn | Mua mới khoảng ~75.000đ | Bất kỳ bản clone nào cũng chạy được; giả định các giới hạn của chip gốc là 32 KB flash / 2 KB SRAM trong firmware. |
| Động cơ bước | Động cơ bước NEMA17 (NEMA17HS3401S) | 130.000đ | [lkcg.vn](https://lkcg.vn/dong-co-step-size-42-nema17hs3401s) | Thân động cơ ngắn/nhỏ gọn phù hợp với thiết kế của rig. |
| Driver động cơ bước | DRV8825 (hoặc A4988 / TMC2209) | 52.500đ | [lkcg.vn](https://lkcg.vn/mach-dieu-khien-dong-co-buoc-drv8825) | Đặt Vref thành 0.485 V (tương đương giới hạn dòng điện ≈ 0.9 A, bằng 90% dòng định mức của động cơ). |
| Cảm biến mã hóa vòng quay | AS5600 12-bit giao tiếp I²C | 42.525đ | [Shopee](https://vn.shp.ee/SwrcwD42) | Đo góc con lắc. Sử dụng thư viện Arduino RobTillaart. **Lưu ý:** Nam châm đi kèm link Shoppe này có thể nhỏ hơn 6mm, cần điều chỉnh khoảng cách lắp đặt từ ~1-2mm so với mặt cảm biến để đảm bảo cường độ từ trường ổn định. |
| Bộ nguồn adapter | Nguồn 12 V, 2 A jack tròn (5.5 × 2.1 mm) | Đã có sẵn | Mua mới khoảng ~50.000đ | Cấp nguồn cho đường 12 V; nguồn 5 V của Arduino được lấy từ IC ổn áp của Nano. |
| Jack nguồn DC | Jack cắm cái gắn bảng (panel-mount socket) 5.5 × 2.1 mm | Đã có sẵn | Mua mới khoảng ~5.000đ | Đầu vào nguồn phía bo mạch. |
| Công tắc nguồn | Công tắc bập bênh tròn 6A/250V (2 chân, đường kính 20mm) | 8.630đ | [Shopee](https://vn.shp.ee/FWKpHFhS) | Mắc nối tiếp trên đường nguồn 12 V; bật/tắt thiết bị mà không cần rút phích cắm nguồn. |
| Tụ sứ 100 nF | Tụ sứ vàng 104 (100nF 50V) | 7.560đ | [Shopee](https://vn.shp.ee/MXgza1Rs) | Mắc song song qua đường nguồn 12 V gần driver. |
| Tụ hóa 22 µF | Tụ hóa 22µF 25V phân cực | 10.289đ | [Shopee](https://vn.shp.ee/ntHt8A56) | Lọc nguồn dung lượng lớn trên đường 12 V (phân phối theo túi 5 con). |
| Bo mạch lỗ | PCB phíp lỗ FR4 loại tốt (40x60mm) | 13.000đ | [lkcg.vn](https://lkcg.vn/pcb-phip-fr4-loai-tot) | Bo mạch chất lượng cao, hai mặt lỗ mạ thiếc dùng để hàn toàn bộ mạch điện tử. |
| Rào cắm (Header pins) | Rào cắm cái (Female), khoảng cách chân 2.54 mm | Đã có sẵn | Mua mới khoảng ~5.000đ | Dùng để gắn Nano + driver dưới dạng các module có thể tháo rời. |
| Dây điện nối mạch | Dây điện Teflon 24AWG [0.22mm²] chịu nhiệt | 20.800đ | [Shopee](https://vn.shp.ee/bSo1jirN) | Mua lẻ 4 mét (Đỏ, Đen, Vàng, Xanh lá). Vỏ Teflon chịu nhiệt tốt, chống cháy và dẫn điện tốt cho nguồn & tín hiệu. |

---

## Linh kiện cơ khí (Mechanical)

| Linh kiện | Thông số (Spec) | Giá (đ) | Nguồn | Ghi chú |
| --------------------- | ---------------------------------------------------------------- | ----------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vòng bi (Bạc đạn) | Vòng bi bạc đạn 608 (độ chính xác cao 8 × 22 × 7 mm) | 6.999đ | [Shopee](https://vn.shp.ee/3EJpdA2M) | Sử dụng một vòng bi duy nhất tại khớp nối cánh tay/con lắc. Loại vòng bi ván trượt cũ hoặc sơ mi nhựa giúp giảm ma sát tối đa và dễ tra dầu mỡ. |
| Các bộ phận in 3D | Cánh tay (Arm), đế (base), nắp (lid), liên kết con lắc (pendulum link), khớp nối cảm biến | 24.000đ (nhựa in) | Tự in hoặc dịch vụ in 3D | Khoảng <100 g nhựa PLA. Các tệp STL nằm trong thư mục `meshes/`; [Mã nguồn OnShape](https://cad.onshape.com/documents/fa8afe5031ca70c78442e408/w/5519455d45464bacd4cf9b1d/e/79273ac76c3305af463951de). Khối lượng, COM và tensor quán tính của liên kết con lắc nằm trong `urdf/model.urdf`. |
| Đồng xu 2p (Anh) | Đồng xu 2 new pence của Anh, nặng 7.12 g | 10.000đ | [Shopee](https://vn.shp.ee/RaSXvYUe) | Lắp chặt (press-fit) vào liên kết con lắc để làm khối lượng đầu mút. Khối lượng, COM và quán tính của nó đã được tính toán sẵn trong mô hình cơ học của con lắc. |
| Tản nhiệt động cơ | Nhôm tản nhiệt 40x40x11mm (Đen/Vàng) | 26.677đ | [Shopee](https://vn.shp.ee/hQRBv1vs) | Dán vào mặt sau của động cơ bước NEMA17 để làm mát trong quá trình hoạt động và huấn luyện kéo dài. |

---

## Phụ kiện & Công cụ hàn mạch (Soldering Accessories & Tools)

| Phụ kiện | Thông số (Spec) | Giá (đ) | Nguồn | Ghi chú |
| ---------------------- | ----------------------------------------------- | --------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bột vệ sinh mũi hàn | Bột phục hồi đầu mỏ hàn (Tip Tinner / Activator) | 25.000đ | [Shopee](https://vn.shp.ee/XoHCHL1Q) | Dùng để phân hủy lớp oxit đen bám trên đầu mỏ hàn rỉ, giúp đầu hàn sáng bóng và ăn thiếc trở lại. |
| Bùi nhùi vệ sinh mũi hàn | Bùi nhùi đồng vệ sinh đầu mỏ hàn chuyên dụng | 15.000đ | [Shopee](https://vn.shp.ee/UTV41c83) | Làm sạch đầu hàn mà không gây xước lớp mạ bảo vệ bên ngoài của đầu hàn. |

### Các công cụ khác (để tham khảo, không tính vào chi phí sản phẩm)

- Mỏ hàn + thiếc hàn (có chì hoặc không chì)
- Máy in 3D (nhựa PLA / PETG, đầu in 0.4 mm, độ dày lớp 0.2 mm)
- Đồng hồ vạn năng (đo thông mạch + tinh chỉnh biến trở Vref trên driver)
