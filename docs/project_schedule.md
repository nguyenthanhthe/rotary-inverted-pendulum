# Kế Hoạch Tiến Độ Dự Án Con Lắc Ngược Quay (Rotary Inverted Pendulum)

*Ngày lập kế hoạch: 20/06/2026*  
*Trạng thái hiện tại:*
*   **Phần mềm**: Đã cấu hình và cài đặt hoàn tất môi trường Python 3.12 (MuJoCo, Stable Baselines 3, PyTorch CUDA GPU) và các thư viện cho Arduino CLI.
*   **Linh kiện đã có**: Tụ sứ 104, tụ hóa 22µF, đồng xu 2-pence (làm tạ đầu mút), công tắc bập bênh Kd105, Arduino Nano, Adapter nguồn 12V 2A, Jack nguồn DC, Rào cắm cái (Female header pins).
*   **Linh kiện đang chờ hoặc chưa đặt**: Động cơ bước NEMA17, Driver DRV8825, Cảm biến AS5600, Phíp lỗ hàn PCB, Vòng bi 608, Tản nhiệt động cơ, các chi tiết in 3D.

---

## 📅 Lộ Trình Tiến Độ Hàng Tuần (4 Tuần)

```mermaid
gantt
    title Lộ Trình Chế Tạo & Huấn Luyện Con Lắc Ngược Quay
    dateFormat  YYYY-MM-DD
    section Tuần 1: Chuẩn bị & In 3D
    Đặt mua linh kiện còn thiếu       :active,   des1, 2026-06-20, 2d
    Đặt dịch vụ hoặc tự in 3D các chi tiết :active, des2, 2026-06-21, 4d
    Chuẩn bị công cụ hàn, dây nối      :active, des3, 2026-06-23, 3d
    section Tuần 2: Hàn mạch & Lắp ráp
    Hàn bo mạch điều khiển (Nano + DRV8825) :todo, des4, 2026-06-28, 3d
    Lắp ráp phần khung cơ khí & Động cơ    :todo, des5, 2026-07-01, 3d
     section Tuần 3: SysID & Đo đạc
    Kiểm tra chiều động cơ & Đọc cảm biến I2C :todo, des6, 2026-07-05, 3d
    Chạy nhận dạng hệ thống (SysID Wizard) :todo, des7, 2026-07-08, 4d
    section Tuần 4: Train & Deploy
    Huấn luyện Sim Curriculum (SAC)         :todo, des8, 2026-07-12, 2d
    Tinh chỉnh Async Fine-tuning trên rig thực :todo, des9, 2026-07-14, 3d
    Chưng cất (Distill) & Nạp Nano độc lập :todo, des10, 2026-07-17, 3d
```

---

## 🛠️ Chi Tiết Nhiệm Vụ Từng Tuần

### Tuần 1: Chuẩn Bị Linh Kiện & In 3D (20/06 - 27/06)
*   **Mục tiêu**: Hoàn tất việc mua sắm phần cứng, in 3D các chi tiết nhựa để có đầy đủ phôi cơ khí khi các linh kiện điện tử về tới nơi.
*   **Các nhiệm vụ cụ thể**:
    1.  **Đặt mua ngay các linh kiện còn thiếu**:
        *   Động cơ bước NEMA17 (khuyên dùng mã `NEMA17HS3401S` thân ngắn để nhẹ rig).
        *   Mạch driver DRV8825 (hoặc A4988).
        *   Cảm biến AS5600 12-bit (chọn link có kèm nam châm 6mm).
        *   Vòng bi bạc đạn 608 (1 cái cho khớp nối con lắc).
        *   Bo mạch lỗ phíp FR4 (40x60mm) và Nhôm tản nhiệt động cơ (40x40x11mm).
    2.  **In 3D các bộ phận nhựa**:
        *   Tải các tệp STL trong thư mục `meshes/` mang đi in dịch vụ hoặc tự in nhựa PLA (Infill nên cài đặt \(\ge 20\%\), 3 đường tường bao - wall lines để bảo đảm độ cứng cơ học).
        *   Các bộ phận gồm: Cánh tay (Arm), liên kết con lắc (Pendulum link), đế (Base), nắp (Lid) và khớp nối cảm biến.
    3.  **Chuẩn bị công cụ**: Mỏ hàn, thiếc hàn, đồng hồ vạn năng (để đo thông mạch và chỉnh Vref của driver).

---

### Tuần 2: Hàn Mạch Điện Tử & Lắp Ráp Cơ Khí (28/06 - 04/07)
*   **Mục tiêu**: Hoàn thành phần cứng vật lý của hệ thống.
*   **Các nhiệm vụ cụ thể**:
    1.  **Hàn bo mạch**:
        *   Hàn rào cắm cái (Female header pins) lên PCB lỗ để có thể cắm/rút Arduino Nano và Driver DRV8825 dễ dàng (đề phòng cháy chập có thể thay thế ngay).
        *   Hàn giắc nguồn DC cái, công tắc bập bênh Kd105 (nối tiếp trên đường 12V cấp cho chân VMOT của driver).
        *   Hàn hai tụ lọc nguồn (tụ hóa 22µF song song tụ sứ 104) sát chân nguồn 12V của Driver để triệt tiêu nhiễu xung dòng điện lúc động cơ đảo chiều.
        *   Hàn đường I2C (SDA/SCL) nối từ Arduino Nano (A4/A5) ra rào cắm cảm biến AS5600.
    2.  **Lắp ráp cơ khí**:
        *   Lắp động cơ NEMA17 vào đế nhựa. Dán tản nhiệt nhôm vào mặt sau động cơ.
        *   Lắp cánh tay quay (Arm) vào trục động cơ bước.
        *   Ép vòng bi 608 vào khớp nối cánh tay và thanh lắc.
        *   Nhét đồng xu 2-pence vào đầu mút thanh lắc làm đối trọng (đã được tính toán khối lượng trong file URDF).
        *   Gắn cảm biến AS5600 đối diện với nam châm (khoảng cách lý tưởng 1.5 - 2 mm).

---

### Tuần 3: Kiểm Tra Tín Hiệu & Nhận Dạng Hệ Thống (05/07 - 11/07)
*   **Mục tiêu**: Đảm bảo tất cả phần cứng giao tiếp chính xác và tạo được file cấu hình động lực học thực tế của máy.
*   **Các nhiệm vụ cụ thể**:
    1.  **Kiểm tra chiều & Căn chỉnh**:
        *   Nạp code test I2C để đảm bảo Arduino nhận được cảm biến AS5600 ở địa chỉ `0x36`.
        *   Xác minh chiều quay: Quay cánh tay sang phải (ngược chiều kim đồng hồ nhìn từ trên xuống) thì vị trí động cơ phải tăng. Nghiêng con lắc sang phải thì góc con lắc phải thay đổi đúng chiều.
        *   **Chỉnh Vref**: Dùng đồng hồ vạn năng đo và vặn biến trở trên DRV8825 để đạt mức Vref khoảng **0.485V** (giới hạn dòng điện \(\approx 0.9A\) để motor khỏe mà không quá nóng).
    2.  **Chạy System Identification (SysID)**:
        *   Nạp file `LowLevelServer.ino` lên Arduino Nano.
        *   Kích hoạt môi trường Conda và chạy bộ Wizard nhận dạng hệ thống:
          ```powershell
          conda activate rotary-inverted-pendulum
          cd RotaryInvertedPendulum-python/src/rl
          python sysid_wizard.py
          ```
        *   Wizard sẽ tự chạy cánh tay để đo thời gian tăng tốc động cơ (\(\tau\)) và chu kỳ lắc tự do để cập nhật file `sysid_params.json`.

---

### Tuần 4: Huấn Luyện RL & Triển Khai Chạy Độc Lập (12/07 - 19/07)
*   **Mục tiêu**: Đưa con lắc tự thăng bằng độc lập bằng mạng nơ-ron không cần cắm máy tính.
*   **Các nhiệm vụ cụ thể**:
    1.  **Huấn luyện Curriculum trong mô phỏng**:
        *   Chạy tập lệnh bash để huấn luyện SAC qua 3 giai đoạn có xáo trộn miền (Domain Randomization):
          ```powershell
          bash curriculum_train.sh
          ```
    2.  **Tinh chỉnh trên máy thực (Async Fine-Tuning)**:
        *   Chạy tinh chỉnh 50 tập thực tế để bù đắp sai số cơ khí:
          ```powershell
          python finetune_async.py --policy runs/<sim_run>/last.zip --port <PORT> --episodes 50 --run-name async_v1
          ```
    3.  **Chưng cất & nạp code độc lập**:
        *   Chưng cất mô hình giáo viên đã tinh chỉnh thành mô hình học sinh MLP 5KB:
          ```powershell
          python distill.py --teacher runs/async_v1/last.zip --buffer runs/async_v1/replay_buffer.pkl --out-dir runs/async_v1/distill_student
          ```
        *   Xuất trọng số sang header C++:
          ```powershell
          python export_weights.py --student runs/async_v1/distill_student/student.pt --header ../../../RotaryInvertedPendulum-arduino/RLControl/policy_weights.h --source-name async_v1/distill_student
          ```
        *   Mở Arduino IDE biên dịch và nạp chương trình `RLControl.ino` lên Arduino Nano.
        *   Rút dây cáp USB, cấp nguồn 12V từ adapter và thưởng thức con lắc tự swing-up và thăng bằng hoàn toàn độc lập!
