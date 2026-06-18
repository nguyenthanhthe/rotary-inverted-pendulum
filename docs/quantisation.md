# Lượng tử hóa trọng số trên thiết bị (On-device weight quantisation)

Các ghi chú về việc lượng tử hóa mô hình mạng distilled student MLP cho Arduino Nano. Cơ sở lý thuyết rộng hơn (khi nào cần bận tâm, quy trình từ đầu đến cuối, phạm vi thay đổi mã nguồn) nằm trong [`policy_improvement_ideas.md`](policy_improvement_ideas.md) phần "Hiệu năng trên thiết bị"; tệp này trình bày về **lựa chọn định dạng** và các kỹ thuật thực tế để chạy được định dạng int8.

Arduino Nano sử dụng vi điều khiển **AVR** ATmega328P. AVR là dòng vi điều khiển RISC 8-bit ban đầu được thiết kế bởi Alf-Egil Bogen và Vegard Wollan (do đó có tên "AVR" — *Alf and Vegard's RISC*) và hiện được bán bởi Microchip Technology sau khi mua lại Atmel. AVR không có bộ tính toán số thực cứng FPU, vì vậy việc lựa chọn định dạng trọng số phụ thuộc hoàn toàn vào các phép nhân số nguyên mà phần cứng có thể thực hiện nguyên bản.

## int8 so với int16 trên AVR

So sánh tốc độ và độ chính xác:

| Định dạng | Số byte/trọng số | Chi phí mỗi phép MAC | Độ phân giải lượng tử hóa | Lượt truyền xuôi (Forward pass - 1216 MAC) |
|---|---|---|---|---|
| float32 | 4 | ~190 chu kỳ (MUL + ADD, mô phỏng bằng phần mềm) | ~7 chữ số thập phân | ~14.5 ms |
| **int8** | **1** | **~5 chu kỳ** (1× MUL + tích lũy) | **max(W) / 127** | **~0.4 ms (~nhanh hơn 36 lần)** |
| int16 | 2 | ~15 chu kỳ (4× MUL + tích lũy) | max(W) / 32767 (~mịn hơn 256 lần so với int8) | ~1.1 ms (~nhanh hơn 13 lần) |

AVR có bộ nhân phần cứng `MUL` (8×8→16) và `MULS` (8x8 có dấu) nhưng **không có bộ nhân 16×16 phần cứng** — phép nhân int16 được cấu thành từ bốn phép nhân 8×8 MUL cộng với các phép cộng nhớ (carries). Do đó, int16 chậm hơn 3 lần trên mỗi phép MAC so với int8 dù số lượng byte chỉ tăng gấp đôi.

## Quyết định

**Sử dụng int8.** Đường dẫn phần cứng là trực tiếp (một lệnh `MULS` duy nhất cho mỗi tích trọng số × kích hoạt, mất hai chu kỳ), và tốc độ tăng 36 lần so với float32 là lý do chính để thực hiện lượng tử hóa ngay từ đầu. Huấn luyện nhận biết lượng tử hóa (Quantisation-aware training - QAT) kết hợp với các kỹ thuật bên dưới giúp thu hẹp khoảng cách độ chính xác xuống mức không đáng kể — đây cũng là quy trình chuẩn mà hệ sinh thái NN nhúng (TFLite Micro, ONNX Runtime Mobile, các bài báo MobileNet) chuẩn hóa.

**int16 là phương án dự phòng** nếu int8 không thể đạt được hành vi cân bằng vòng kín của mô hình float student. int16 thậm chí với lượng tử hóa sau huấn luyện (post-training quantisation) về cơ bản có độ chính xác tương đương với số thực float (độ phân giải mịn hơn 256 lần so với int8), do đó nó là một lưới bảo hiểm an toàn — nhưng với chi phí MAC cao gấp 3 lần, nó không nên là lựa chọn mặc định.

## Những gì cần thiết để int8 thực sự hoạt động trên thiết bị này

Một pipeline int8 + QAT ngây thơ (chỉ nhân tỉ lệ cho mỗi tensor ở mọi nơi, trọng số và kích hoạt đều nằm trong phạm vi int8, không xử lý đặc biệt cho bias hoặc tính không đồng nhất của đầu vào) sẽ bị kẹt ở **điểm số trung bình thẳng đứng ~0.70** trên thiết bị này — chính sách *bắt* được con lắc một cách đáng tin cậy nhưng không thể *giữ* nó thẳng đứng, và sẽ trôi khỏi điểm thu hút tĩnh chỉ sau vài giây. Để thu hẹp khoảng cách đó lên mức 0.951 yêu cầu phải áp dụng kết hợp nhiều kỹ thuật bổ sung lên trên QAT thông thường. Không có kỹ thuật nào là kỳ dị; tất cả đều là các mẹo kinh điển để "làm cho int8 hoạt động".

### 1. Huấn luyện nhận biết lượng tử hóa (QAT)

Huấn luyện mô hình với phép làm tròn int8 giả lập được chèn vào lượt truyền xuôi (forward pass) trong quá trình huấn luyện. Bộ tối ưu hóa (optimizer) sau đó sẽ học các trọng số *có khả năng chống chịu với sai số làm tròn do int8 gây ra*, thay vì bị bất ngờ bởi chúng khi triển khai thực tế.

Cơ chế giả lập là "fake-quant + straight-through estimator": lượt truyền xuôi làm tròn các giá trị kích hoạt (activations) và trọng số (weights) về lưới int8, lượt lan truyền ngược (backward pass) giả vờ rằng phép làm tròn đó là hàm đồng nhất (identity) để các gradient lan truyền bình thường.

### 2. Lượng tử hóa đầu vào theo từng kênh (Per-channel input quantisation)

Năm chiều quan sát (obs dimensions) có các phạm vi giá trị cực kỳ khác nhau:

| Chiều quan sát | Phạm vi | LSB ở thang đo chung | LSB ở thang đo từng kênh |
|---|---|---|---|
| `motor_pos` | ±2.2 rad | 0.23 (12 giá trị phân biệt) | 0.014 (164 giá trị phân biệt) |
| `sin(theta)` | ±1.0 | 0.23 (8 giá trị phân biệt) | 0.008 (256 giá trị phân biệt) |
| `cos(theta)` | ±1.0 | 0.23 (8 giá trị phân biệt) | 0.008 (256 giá trị phân biệt) |
| `motor_vel` | ±15 rad/s | 0.23 | 0.029 |
| `pen_vel` | ±30 rad/s | 0.23 | 0.229 |

Một thang đo quan sát *chung (shared)*, được thiết lập bởi chiều có phạm vi lớn nhất (`pen_vel` đạt tới ±30 rad/s trong quá trình swing-up), sẽ đè bẹp `sin(theta)` và `cos(theta)` xuống chỉ còn 4 giá trị int8 phân biệt ngay tại điểm cân bằng nơi chúng cần độ chính xác tối đa. Việc cung cấp cho mỗi chiều quan sát một thang đo **riêng** của nó giúp khôi phục độ mịn LSB gấp ~30 lần trên các chiều có phạm vi nhỏ mà không làm mất phạm vi hoạt động của các chiều có phạm vi lớn. Đây là cải tiến quan trọng nhất — tạo nên sự khác biệt giữa "không thể giữ cân bằng" và "cân bằng tốt".

### 3. Lượng tử hóa trọng số theo từng hàng (Per-row weight quantisation)

Đối với mỗi lớp Tuyến tính (Linear layer), hãy cung cấp cho mỗi hàng trọng số của neuron đầu ra thang đo *riêng* của nó thay vì chia sẻ một thang đo duy nhất trên toàn bộ ma trận trọng số. Các neuron đầu ra khác nhau có độ lớn trọng số khác nhau, và việc ép chúng chia sẻ một thang đo chung sẽ lãng phí độ chính xác trên các hàng có giá trị cực đại thấp hơn nhiều so với giá trị cực đại toàn cục. Đây là mẫu thiết kế tiêu chuẩn của TFLite-Micro.

Việc đổi thang đo (rescale) cho đầu vào int8 của lớp tiếp theo cũng trở thành theo từng kênh đầu ra — thay vì một hệ số nhân dấu phẩy cố định duy nhất `M_q15`, Arduino sẽ có một mảng các hệ số này, được tra cứu cho từng neuron đầu ra trong quá trình matmul.

### 4. Lượng tử hóa bias thành int32, không phải int8

Các bias nằm trong bộ tích lũy int32 (accumulator) trên đường dẫn triển khai thực tế, nơi chúng được cộng vào `sum_j (W_int8[i,j] × x_int8[j])` trước khi đổi thang đo. "Kích thước bước" mà bias làm tròn tới là `s_w × s_x` (đơn vị của bộ tích lũy), nhỏ hơn nhiều so với 1 — các giá trị bias thông thường là bội số nguyên của `s_w × s_x` trong khoảng hàng *nghìn*.

Lần thử nghiệm đầu tiên đã sử dụng cùng một kiểu fake-quant int8 cho bias như đối với trọng số, điều này **giới hạn chúng trong khoảng ±127 × s_w × s_x**. Đối với lớp thứ 3 thông thường có `s_w × s_x ≈ 6e-4`, một giá trị bias là 0.1 → được lượng tử hóa thành 170 → bị giới hạn về 127 → mất đi 25% giá trị của nó. Điều này gây ra lỗi nghiêm trọng. Giải pháp khắc phục là sử dụng một bộ "int32 fake-quant" riêng biệt giúp làm tròn về lưới nhưng không giới hạn (chúng tôi sử dụng ±2³⁰ làm giới hạn an toàn; các bias trong thực tế cùng lắm chỉ ở mức O(10²)).

### 5. Bỏ qua lượng tử hóa trước tanh (Skip pre-tanh quantisation)

Kích hoạt của lớp đầu ra đi qua hàm `tanh`. Lần thử nghiệm đầu tiên đã lượng tử hóa giá trị trước tanh thành int8 (khớp với những gì một triển khai dựa trên bảng tra cứu LUT giả định). Nhưng đường dẫn triển khai thực tế không sử dụng bảng LUT — nó giải lượng tử hóa (dequantise) bộ tích lũy int32 trực tiếp thành float và gọi hàm thư viện `tanhf` của libm. Do đó, QAT cũng không nên lượng tử hóa trước tanh; việc làm này chỉ huấn luyện các trọng số bù đắp cho một phần mất mát độ chính xác mà thực tế triển khai không bao giờ gặp phải, khiến kết quả tệ hơn.

Hàm `tanh` được gọi một lần cho mỗi lượt suy luận trên AVR (mất khoảng ~200 chu kỳ), do đó một lớp cuối cùng dựa trên số thực float về cơ bản là miễn phí.

### 6. Hấp thụ Lớp 1 (Layer-1 absorbing)

Lượng tử hóa đầu vào theo từng kênh (mẹo 2) cung cấp cho mỗi chiều quan sát thang đo riêng `s_obs[j]`. Lượng tử hóa trọng số theo từng hàng (mẹo 3) cung cấp cho mỗi hàng đầu ra thang đo riêng `s_w[i]`. Khi kết hợp lại, phép nhân ma trận `y = Wx + b` được mở rộng thành:

```
y[i] = sum_j (s_w[i] × W_int[i,j]) × (s_obs[j] × x_int[j])
     = sum_j  s_w[i] × s_obs[j] × W_int[i,j] × x_int[j]
```

Mỗi số hạng có một thang đo *khác nhau* (`s_w[i] × s_obs[j]`), do đó chúng ta không thể nhóm một hệ số nhân chung và áp dụng nó một lần sau khi tính tổng. Điều đó sẽ phá vỡ mẫu thiết kế đơn giản "nhân ma trận số nguyên + đổi thang đo theo hàng" giúp int8 chạy nhanh trên AVR.

Giải pháp: tại *thời điểm xuất mô hình* (chứ không phải lúc runtime), hãy nhân trước các trọng số với thang đo đầu vào của từng kênh tương ứng:

```
W_eff[i,j] = W[i,j] × s_obs[j]                ← các cột được kéo giãn theo thang đo đầu vào của chúng
W_eff_int[i,j] = round(W_eff[i,j] / s_w_eff[i])   ← sau đó lượng tử hóa theo từng hàng
```

Khi đó Arduino chỉ cần chạy phép nhân ma trận int8 thông thường + đổi thang đo theo hàng:

```
accum[i] = sum_j  W_eff_int[i,j] × x_int[j]
y[i]     = accum[i] × s_w_eff[i]
```

Kết quả trả về là như nhau, nhưng thang đo đầu vào của từng kênh đã được "hấp thụ" vào ma trận trọng số. Arduino không bao giờ nhìn thấy `s_obs` trực tiếp khi chạy; thông tin đó đã được tích hợp vào `W_eff_int` tại thời điểm biên dịch. Điều này giúp tiết kiệm tài nguyên khi chạy, chỉ cần thêm một chút xử lý trong tệp `export_weights_quantised.py`.

## Kết quả cuối cùng

Thử nghiệm từ đầu đến cuối trên thiết bị thực với tất cả 6 mẹo được áp dụng cùng nhau:

| Phiên bản | val_mse | Trung bình thẳng đứng (tethered) |
|---|---|---|
| Float H=16 (tham chiếu sản xuất) | 0.040 | 0.946 |
| Naive int8 H=16 (lượng tử hóa chung cho mọi thứ) | 0.080 | 0.706 |
| Naive int8 H=32 (tăng dung lượng, cùng mức nhiễu) | 0.088 | 0.696 |
| **Per-channel int8 H=16** (đầy đủ các mẹo) | **0.045** | **0.934** |
| **Per-channel int8 H=32** (đầy đủ các mẹo) | **0.040** | **0.951** |

**Sản xuất:** Phiên bản int8 H=32 với QAT + các mẹo 1–6, đạt điểm số trung bình thẳng đứng 0.951 khi cắm máy tính — nằm trong phạm vi nhiễu ngẫu nhiên giữa các lần thử so với baseline float, tốc độ suy luận đạt ~0.4 ms trên AVR (so với ~15 ms của float). Kích thước trọng số chỉ 1.5 KB (nhỏ hơn 4 lần so với float). Phiên bản int8 trên thiết bị là lộ trình triển khai chạy độc lập trong tương lai. Bản build float (khi bỏ định nghĩa `#define POLICY_QUANTISED`) được giữ lại làm tham chiếu so sánh.

Một khác biệt có thể quan sát được khi triển khai: học sinh int8 có xu hướng rơi vào điểm thu hút "hiệu chỉnh tích cực" (dao động nhỏ nhưng liên tục của động cơ khi cân bằng) trong khi học sinh float thường rơi vào điểm thu hút "tĩnh" (động cơ cơ bản đứng yên ở một góc lệch cố định). Cả hai đều cân bằng tốt như nhau; lựa chọn này bị chi phối bởi nhiễu huấn luyện SAC — xem [`control_rate_selection.md`](control_rate_selection.md) để biết động lực học nền tảng của việc phân chia điểm thu hút này.

## Tại sao cần bận tâm — int8 thực sự mang lại điều gì

Một câu hỏi hợp lý khi đặt chúng cạnh nhau: phiên bản float H=16 đã cân bằng ở mức 0.95 với hành vi động cơ êm hơn, vậy đường dẫn int8 nhanh hơn 36 lần mang lại lợi ích gì cho thiết bị *này*?

Đối với tác vụ hiện tại (cân bằng + swing-up ở tần số 35 Hz với các quan sát Markovian hiện có), hầu như không mang lại lợi ích gì — phiên bản float H=16 có thừa khoảng dự phòng về độ trễ và điểm thu hút tĩnh của nó tốt hơn về mặt chất lượng so với hiệu chỉnh tích cực của học sinh int8. Hai lý do thực tế khiến lượng tử hóa vẫn quan trọng ở đây là:

### 1. Các đầu vào lớn hơn và kiến trúc mạng phong phú hơn không bị giới hạn bởi độ trễ

Học sinh float H=16 mất ~5 ms cho mỗi lượt suy luận, nằm gọn trong quỹ thời gian 28.6 ms của tick điều khiển. Hầu hết các mở rộng mạng mà chúng ta muốn thử nghiệm sau này sẽ đẩy con số đó sát hoặc vượt quá quỹ thời gian này:

| Thay đổi kiến trúc | Suy luận Float | Suy luận Int8 |
|---|---|---|
| Xếp chồng khung hình (Frame stacking) (N=4 → 20 chiều quan sát, MLP H=32) | ~7 ms | ~0.3 ms |
| GRU H=16 (lớp tuần hoàn) | ~25 ms | ~1 ms |
| MLP H=64 (mạng truyền thẳng phong phú hơn) | ~50 ms | ~1.5 ms |

**Cảnh báo trung thực về tiềm năng**: chúng ta không thể khẳng định chắc chắn rằng bất kỳ mở rộng nào trong số này *sẽ* cải thiện hiệu suất thiết bị. Học sinh float hiện tại đã đạt điểm số 0.95+ khi cắm máy tính, mức độ này có khả năng đã chạm trần nhiễu phần cứng (AS5600 12-bit ≈ độ phân giải 0.09°, lượng tử hóa vi bước AccelStepper ≈ 0.225°, quá trình chuyển đổi của vòng bi). Mạng GRU *có thể* học cách tự hiệu chuẩn độ trôi lệch của encoder, nhưng thiết bị thực tế không cho thấy dấu hiệu bị ảnh hưởng nghiêm trọng bởi lỗi này. Xếp chồng khung hình *có thể* làm mịn các ước tính vận tốc, nhưng đầu vào hiện tại đã bao gồm vận tốc được lọc. Việc các cải tiến này có mang lại hiệu năng tốt hơn đo đạc được hay không chỉ là hy vọng, chưa được chứng minh.

Vì vậy, nhận định trung thực là: int8 là **cơ sở hạ tầng để không ngăn cản các thử nghiệm trong tương lai**, chứ không phải cơ sở hạ tầng đảm bảo cải thiện hiệu năng ngay hôm nay. Vào ngày chúng ta muốn thử xếp chồng khung hình (~một ngày làm việc) hoặc mạng GRU (sẽ mất nhiều ngày di chuyển framework vì SB3 không đi kèm SAC tuần hoàn — sẽ cần dùng CleanRL hoặc Tianshou), quỹ thời gian suy luận đã có sẵn ở đó. Nếu không có int8, cả hai thử nghiệm này đều bị chặn bởi thời gian suy luận ngay cả trên chính sách siêu nhỏ này.

### 2. Giá trị học tập / hồ sơ dự án (portfolio)

Một pipeline int8-chạy-trên-AVR hoạt động hoàn chỉnh chính là mô hình thu nhỏ của quy trình triển khai ML di động chuẩn — một điểm cộng lớn cho năng lực dự án và là điểm đích tự nhiên cho hành trình khám phá xem công nghệ này có thể đi sâu đến mức nào.

## Tại sao người ta cần lượng tử hóa (bối cảnh rộng hơn)

Đối với thiết bị *này*, tốc độ tăng của int8 là trường hợp hiếm hoi lợi ích thể hiện trực tiếp ở thời gian tính toán, bởi vì chip ATmega328P không có FPU — mỗi phép nhân số thực float đều phải giả lập bằng phần mềm, do đó đường dẫn nhân phần cứng của int8 nhanh hơn khoảng 36 lần. Trên phần cứng hiện đại (điện thoại, GPU máy chủ, hoặc thậm chí Raspberry Pi), tính toán số thực float rất nhanh và tốc độ tăng của int8 thể hiện tinh tế hơn. Tuy nhiên, kỹ thuật này vẫn xuất hiện ở *mọi nơi* trong môi trường chạy thực tế của ML, bởi vì ba yếu tố khác bắt đầu chiếm ưu thế:

- **Băng thông bộ nhớ (Memory bandwidth), chứ không phải tính toán, là điểm nghẽn trên các bộ tăng tốc.** Một card đồ họa A100 có thể thực hiện ~10 TFLOPS fp32 nhưng chỉ lấy được ~2 TB/s dữ liệu từ bộ nhớ HBM. Một mô hình ngôn ngữ lớn LLM 70 tỷ tham số (70B) ở định dạng fp32 (dung lượng 280 GB) sẽ mất ~140 ms *chỉ để tải các trọng số qua bộ nhớ một lần*. Ở định dạng int4 (35 GB), lượt tải tương tự chỉ mất ~17 ms. Phép tính số học không rẻ hơn, nhưng việc nạp dữ liệu cho chip thì nhanh hơn — và đó hầu như luôn là ràng buộc chính.
- **Năng lượng và tuổi thọ pin.** Một phép nhân int8 tiêu thụ năng lượng ít hơn khoảng 4 lần so với phép nhân fp32 (diện tích silicon + hoạt động chuyển mạch). Đối với các trung tâm dữ liệu phục vụ hàng tỷ lượt suy luận mỗi ngày, điều này giúp tiết kiệm hàng triệu bảng tiền điện. Đối với các thiết bị chạy bằng pin, nó tạo ra sự khác biệt giữa một giờ và một ngày sử dụng suy luận.
- **Các đường dẫn phần cứng chuyên dụng.** Nhân Tensor Cores của NVIDIA, Neural Engine của Apple, hay TPUs của Google đều có các đường dẫn int8 / int4 / fp8 nhanh hơn *nhiều* so với fp32 — thông thường gấp 4–8 lần hiệu năng. Lý do duy nhất các mô hình LLM trên thiết bị (điện thoại chạy Llama-3, MacBook chạy Mistral) tồn tại là nhờ lượng tử hóa ánh xạ mô hình vào các đường dẫn nhanh của phần cứng chip. Nếu không có int4, một mô hình 70B thậm chí còn không vừa bộ nhớ RAM của một chiếc MacBook 64 GB ngay từ đầu.

Vì vậy, trên một vi điều khiển nhỏ, lợi ích thu được là **thời gian tính toán** (không có FPU); trên điện thoại là **năng lượng và kích thước bộ nhớ**; trên điện toán đám mây là **tối ưu hóa chi phí ($$/token) và độ trễ**. Kỹ thuật áp dụng là như nhau; ràng buộc thực tế sẽ thay đổi theo phần cứng:

| Nơi chạy mô hình | Những gì thực sự được tiết kiệm | Tại sao người ta lượng tử hóa |
|---|---|---|
| **Arduino Nano (chúng ta)** | Chu kỳ nhân số thực float giả lập bằng phần mềm | Khả năng tính toán là ràng buộc chính |
| **Điện thoại chạy Llama-3** | Bộ nhớ RAM (trọng số nhỏ hơn 4 lần) + hiệu năng Neural Engine | Một mô hình 70B fp32 = 280 GB. Không thể chạy được nếu không lượng tử hóa. |
| **Đám mây phục vụ lớp GPT** | Băng thông bộ nhớ (lượt lấy dữ liệu HBM), chi phí $$/token | int4/int8 → tạo ra nhiều token hơn từ 4–8 lần trên mỗi giờ chạy GPU |
| **Thiết bị biên (flycam, chuông cửa, đồng hồ)** | Tuổi thọ pin | Phép nhân int8 sử dụng năng lượng ít hơn khoảng ~4 lần so với fp32 |

Thiết bị của chúng ta là trường hợp hiếm hoi mà lợi ích thể hiện trực tiếp nhất — nhưng cùng một công thức (QAT, thang đo từng kênh, hấp thụ trọng số) chính là những gì đang chạy Llama-3 trên điện thoại, chỉ là ở một quy mô lớn hơn nhiều.
