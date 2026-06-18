# Danh sách linh kiện (Bill of Materials - BOM)

Các linh kiện cần thiết để chế tạo một bộ con lắc ngược quay. Cập nhật lần cuối ngày 25/05/2026.

**Ước tính chi phí**  
Một bộ hoàn chỉnh có chi phí linh kiện **dưới £20** (~£14 linh kiện điện tử + ~£5 linh kiện cơ khí) — rẻ hơn 230 lần so với bộ Quanser QUBE có giá £4,500.

**Cơ sở thiết kế**  
Để hiểu rõ *tại sao* mỗi linh kiện điện tử được chọn (tính toán nguồn điện, chọn động cơ, so sánh các driver, tụ lọc nguồn, v.v.), hãy xem tài liệu [`electronics_design.md`](electronics_design.md). Bản BOM này thuần túy là tài liệu tham khảo để mua sắm.

**Nhà cung cấp**  
Trừ khi có ghi chú khác, các mặt hàng được mua từ AliExpress. Ngoại lệ duy nhất là vòng bi Bones Reds (mua từ Amazon UK).

**Lựa chọn phiên bản (Variant selection)**  
Một số liên kết trên AliExpress bán nhiều phiên bản khác nhau trên cùng một trang sản phẩm (ví dụ: vòng bi ZZ so với RS, chiều dài thân động cơ bước, màu sắc/thông số công tắc, có hoặc không có phụ kiện đi kèm). Lựa chọn mặc định của trang **không phải lúc nào** cũng đúng — hãy luôn kiểm tra cột **Thông số (Spec)** bên dưới và chọn đúng phiên bản trước khi thêm vào giỏ hàng.

## Linh kiện điện tử (Electronics)

| Linh kiện | Thông số (Spec) | Giá (£) | Nguồn | Ghi chú |
| ---------------------- | ----------------------------------------------- | --------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vi điều khiển | Arduino Nano (ATmega328P, 16 MHz, CH340, USB-C) | £1.60 | [AliExpress](https://www.aliexpress.com/item/1005006053215107.html) | Bất kỳ bản clone nào cũng chạy được; giả định các giới hạn của chip gốc là 32 KB flash / 2 KB SRAM trong firmware. |
| Động cơ bước | Động cơ bước NEMA17 17HS4023, dòng định mức 1 A, thân dài 22 mm | £4.48 | [AliExpress](https://www.aliexpress.com/item/1005006111249881.html) | Phiên bản thân ngắn. |
| Driver động cơ bước | DRV8825 (hoặc A4988 / TMC2209) | £1.43 | [AliExpress](https://www.aliexpress.com/item/10000278156894.html) | Đặt Vref thành 0.485 V (tương đương giới hạn dòng điện ≈ 0.9 A, bằng 90% dòng định mức của động cơ). |
| Cảm biến mã hóa vòng quay | AS5600 12-bit giao tiếp I²C (đi kèm nam châm vĩnh cửu phân cực hướng kính - diametric magnet) | £1.04 | [AliExpress](https://www.aliexpress.com/item/1005006349632569.html) | Đo góc con lắc. Sử dụng thư viện Arduino RobTillaart. Module đi kèm với một đĩa nam châm nhỏ phân cực hướng kính gắn ở đầu trục con lắc đối diện với mặt cảm biến AS5600. |
| Bộ nguồn adapter | Nguồn 12 V, 2 A jack tròn (5.5 × 2.1 mm, phích cắm UK) | £2.59 | [AliExpress](https://www.aliexpress.com/item/1005006467110035.html) | Cấp nguồn cho đường 12 V; nguồn 5 V của Arduino được lấy từ IC ổn áp của Nano. |
| Jack nguồn DC | Jack cắm cái gắn bảng (panel-mount socket) 5.5 × 2.1 mm | £0.12 | [AliExpress](https://www.aliexpress.com/item/1005003324016159.html) | Đầu vào nguồn phía bo mạch. |
| Công tắc nguồn | Công tắc bập bênh SPST, tròn 20 mm, 12 V DC | £1.24 | [AliExpress](https://www.aliexpress.com/item/1005005944839290.html) | Mắc nối tiếp trên đường nguồn 12 V; bật/tắt thiết bị mà không cần rút phích cắm. |
| Tụ gốm 100 nF | Tụ gốm nguyên khối (monolithic ceramic) 104 | £0.01 | [AliExpress](https://www.aliexpress.com/item/1005005691676032.html) | Mắc song song qua đường nguồn 12 V gần driver. |
| Tụ hóa 22 µF | Tụ hóa nhôm (aluminium electrolytic) 25 V | £0.02 | [AliExpress](https://www.aliexpress.com/item/1005005945738204.html) | Tụ lọc nguồn dung lượng lớn trên đường 12 V (decoupling). |
| Bo mạch lỗ | Protoboard song song hai mặt 40 × 60 mm | £0.26 | [AliExpress](https://www.aliexpress.com/item/1005005945712659.html) | Toàn bộ linh kiện điện tử được hàn trên này. |
| Rào cắm (Header pins) | Rào cắm cái (Female), khoảng cách chân 2.54 mm, thanh 1×40 | £1.12 | [AliExpress](https://www.aliexpress.com/item/1005006034877497.html) | Để gắn Nano + driver dưới dạng các module có thể tháo rời. |

## Linh kiện cơ khí (Mechanical)

| Linh kiện | Thông số (Spec) | Giá (£) | Nguồn | Ghi chú |
| --------------------- | ---------------------------------------------------------------- | ----------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vòng bi (Bạc đạn) | **Bones Reds 608** (vòng bi trượt ván độ chính xác cao 8 × 22 × 7 mm) | £2.50 | Amazon UK | Sử dụng một vòng bi duy nhất tại khớp nối cánh tay/con lắc, sau khi đơn giản hóa thiết kế hai vòng bi trước đó. Vòng bi ván trượt được thiết kế để có ma sát thấp, dễ dàng tháo nắp chắn bụi và tra lại dầu mỡ. |
| Vòng bi (giá rẻ) | 608ZZ 8 × 22 × 7 mm (có nắp chắn bụi bằng sắt — **tránh phiên bản 608RS**) | £0.18 | [AliExpress](https://www.aliexpress.com/item/1005005778152535.html) | Lựa chọn thay thế giá rẻ có thể dùng được. Cùng một trang bán hàng này cũng bán phiên bản 608RS (phớt cao su) — vòng bi RS có lực cản/ma sát tĩnh (stiction) lớn hơn rõ rệt và nên tránh dùng. Ký hiệu ABEC-7 trên các lô hàng giá rẻ không có nhiều ý nghĩa; vòng bi trượt ván chính xác (Bones Reds hoặc tương đương) vẫn được ưu tiên để có ma sát thấp nhất. |
| Các bộ phận in 3D | Cánh tay (Arm), đế (base), nắp (lid), liên kết con lắc (pendulum link - có chèn đồng xu bên trong), khớp nối cảm biến (encoder boss) | ~£2.00 (nhựa in) | — | Tự in. <100 g nhựa PLA với giá khoảng ~£20/kg. Các tệp STL nằm trong thư mục `meshes/`; [Mã nguồn OnShape](https://cad.onshape.com/documents/fa8afe5031ca70c78442e408/w/5519455d45464bacd4cf9b1d/e/79273ac76c3305af463951de). Toàn bộ cụm chi tiết được thiết kế để **lắp chặt (press-fit)** — không cần ốc vít. Khối lượng, COM và tensor quán tính của liên kết con lắc nằm trong `urdf/model.urdf` (được xuất từ Onshape với các vật liệu đã được hiệu chỉnh mật độ PLA) — cả môi trường mô phỏng RL và pipeline sysid đều đọc dữ liệu từ đây. |
| Đồng xu 2p (Anh) | Đồng xu 2-pence của Anh, 7.12 g, kích thước 25.91 × 2.03 mm | £0.02 | — | Lắp chặt (press-fit) vào liên kết con lắc để làm khối lượng đầu mút. Khối lượng, COM và quán tính của nó được tích hợp sẵn vào thông số quán tính của liên kết con lắc được xuất từ Onshape — xem `urdf/model.urdf`. |
| Tản nhiệt động cơ | Nhôm định hình, phù hợp với động cơ NEMA17 / 42 mm | £0.55 | [AliExpress](https://www.aliexpress.com/item/4000723868050.html) | Dán vào mặt sau của động cơ. Giúp động cơ luôn mát trong các phiên huấn luyện kéo dài. |

## Công cụ (để tham khảo, không tính vào chi phí sản phẩm)

- Mỏ hàn + thiếc hàn 60/40 có chì (hoặc không chì tùy chọn)
- Máy in 3D (nhựa PLA / PETG, đầu in 0.4 mm, độ dày lớp 0.2 mm)
- Đồng hồ vạn năng (đo thông mạch + tinh chỉnh biến trở Vref)
- [Bộ dây lõi đơn 30 AWG](https://www.amazon.co.uk/gp/product/B0C2Z4FNN5) (bộ 5 cuộn, Amazon UK)
