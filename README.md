# Con lắc ngược quay (Rotary Inverted Pendulum)

[![Xem video lắp ráp](assets/youtube-thumbnail-pendulum-build.jpg)](https://www.youtube.com/watch?v=rKChjuuR7K8)

Một hệ con lắc ngược quay (rotary inverted pendulum) tự chế (DIY) bạn có thể tự in, hàn và huấn luyện tại nhà — với chi phí linh kiện khoảng **£20**. Đây là một phiên bản mở, có thể can thiệp phần cứng/phần mềm (hackable) dựa trên các thiết bị mà bạn thường phải mua từ các nhà cung cấp thiết bị phòng thí nghiệm (như QUBE Servo 2 của Quanser có giá khoảng £4,500). Con lắc tự cân bằng bằng một chính sách học tăng cường (reinforcement-learning policy) được huấn luyện trong mô phỏng (simulation), tinh chỉnh trên phần cứng thực tế (real hardware), và lượng tử hóa (quantized) để chạy trên Arduino Nano.

## Có gì trong kho lưu trữ này

| Thư mục | Nội dung |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| [`meshes/`](meshes), [`urdf/`](urdf) | Các tệp STL để in 3D và mô hình URDF (nguồn chân lý duy nhất cho hình học của con lắc) |
| [`diagrams/`](diagrams) | Sơ đồ đấu dây và ảnh linh kiện |
| [`RotaryInvertedPendulum-arduino/`](RotaryInvertedPendulum-arduino) | Firmware — server cấp thấp, bộ điều khiển PID tinh chỉnh thủ công, bộ điều khiển RL chạy trên thiết bị |
| [`RotaryInvertedPendulum-python/`](RotaryInvertedPendulum-python) | Môi trường mô phỏng (sim env), huấn luyện SAC, nhận dạng hệ thống (system identification), cầu nối phần cứng thực, chưng cất (distillation) và xuất int8 |
| [`RotaryInvertedPendulum-julia/`](RotaryInvertedPendulum-julia) | Thử nghiệm MPC/LQR và trực quan hóa MeshCat |
| [`docs/`](docs) | Tài liệu hướng dẫn chế tạo (runbook), BOM (danh sách linh kiện), thiết kế điện tử, tài liệu về stack RL |

## Bắt đầu từ đâu

- **Chế tạo một bộ** — [`docs/BOM.md`](docs/BOM.md), [`docs/electronics_design.md`](docs/electronics_design.md), [`docs/end_to_end_runbook.md`](docs/end_to_end_runbook.md)
- **Huấn luyện một chính sách (policy)** — [`RotaryInvertedPendulum-python/README.md`](RotaryInvertedPendulum-python/README.md) hướng dẫn từng bước về mô phỏng và quy trình huấn luyện
- **Hiểu về stack RL** — [`docs/rl_transitions.md`](docs/rl_transitions.md), [`docs/domain_randomization.md`](docs/domain_randomization.md), [`docs/transport_delay.md`](docs/transport_delay.md), [`docs/quantisation.md`](docs/quantisation.md), [`docs/sysid_runbook.md`](docs/sysid_runbook.md)

## Bạn muốn mua hơn là tự chế?

Các bộ KIT tự chế (DIY kits) có giá khoảng [$100–$200 trên AliExpress](https://www.aliexpress.com/w/wholesale-rotary-inverted-pendulum.html); bộ Quanser QUBE Servo 2 được đề cập ở trên có giá khoảng £4,500.

## Các nghiên cứu/dự án liên quan

- [Desktop Inverted Pendulum, build-its-inprogress](https://build-its-inprogress.blogspot.com/2016/08/desktop-inverted-pendulum-part-2-control.html) ([toàn bộ chuỗi bài viết](https://build-its-inprogress.blogspot.com/search/label/Pendulum))
- [Furuta pendulum, dagor.dev](https://www.dagor.dev/blog/furuta-pendulum)
- [The Rotary Control Lab — Quanser brochure (PDF)](https://tecsolutions.us/sites/default/files/quanser/The%20Rotary%20Control%20Lab%20Brochure_4.pdf)
- [Bài báo khảo sát, *Trans. Inst. Meas. Control*](https://journals.sagepub.com/doi/full/10.1177/00202940211035406)
- Video chế tạo: [[1]](https://www.youtube.com/watch?v=2koXcs0IhOc), [[2]](https://www.youtube.com/watch?v=bY4t6yfBA24), [[3]](https://www.youtube.com/watch?v=VVQ-PGfJMuA)

## Lời cảm ơn

Tôi muốn gửi lời cảm ơn đến những người sau đây vì những đóng góp của họ cho dự án này:
- [Joe](https://github.com/spookycouch) vì đã gợi ý tôi thử học tăng cường với Stable Baselines 3, điều này đã khởi động phần điều khiển học máy của dự án này.
- [Mykha](https://github.com/Mika412) vì những cuộc thảo luận ban đầu về dự án này bên ly bia ở công viên.
- [André](https://github.com/Esser50K), [Rafael](https://github.com/rkourdis), và [Vlad](https://github.com/VladimirIvan) vì các thảo luận kỹ thuật, phản hồi và sự hỗ trợ.
- [Vivek](https://github.com/svrkrishnavivek) vì sự giúp đỡ và phản hồi vô giá của anh ấy về phần điện tử của hệ thống.
- [心诺 (Xinnuo)](https://github.com/XinnuoXu) vì sự đồng hành và hỗ trợ của cô ấy trong khi tôi thực hiện dự án này.

Cuối cùng, tôi muốn cảm ơn cộng đồng mã nguồn mở nói chung vì đã cung cấp các công cụ và tài nguyên giúp dự án này có thể hoàn thành.
