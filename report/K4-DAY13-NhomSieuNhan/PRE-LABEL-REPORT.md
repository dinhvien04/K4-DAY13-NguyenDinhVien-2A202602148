# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm 5 — Nhóm siêu nhân / Phòng Lab Day 13
- Thành viên: xem `TEAMMATES.md` (Nguyễn Đình Viễn - 02148, Lê Đức Minh Quân - 02126, Nguyễn Đức Hà - 02105, Bùi Phương Nam - 02134; vai trò xoay vòng từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Đức Minh Quân (cùng cả nhóm vận hành); 2026-10-01; Linux x86_64 / amd64 (Docker Desktop).
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - Frame: `demo.pcd` (mẫu KITTI 000008 chuyển đổi theo CC BY-NC-SA 3.0)
  - Input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Chạy local offline theo hợp đồng runner `--network none`.
- Checkpoint: PointPillars KITTI có sẵn trong image:
  - Path: `/opt/PointPillars/pretrained/epoch_160.pth`
  - Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (`range: 0.0, -39.68, -3.0, 69.12, 39.68, 1.0`); score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - PCD KITTI Student lược bỏ reflectance thực, sử dụng RGB=0 làm placeholder (kênh hằng số cho adapter).
  - $z_{ground}$ được ước lượng từ đám mây điểm: $z_{ground} = 0.075\text{ m}$.

---

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| **A** | 0 m | 0.16 m | 1 | 0.330 m | `boxes-demo-delta-0-voxel-0.16.json`<br>`side-demo-delta-0-voxel-0.16.png`<br>`summary.csv` | Khi $\Delta z = 0\text{ m}$, mô hình chỉ phát hiện duy nhất **1 hộp** (`vehicles`, score 0.322) ở vị trí $x \approx 13.15, y \approx -0.45, z \approx 0.33$. Đám mây điểm không được bù chiều cao cảm biến nên nằm ngoài phân phối độ cao thông thường của KITTI, dẫn tới bỏ sót 12 đối tượng còn lại. |
| **B** | 1.73 m | 0.16 m | 13 | 1.034 m | `boxes-demo-delta-1.73-voxel-0.16.json`<br>`side-demo-delta-1.73-voxel-0.16.png`<br>`summary.csv` | Khi bù chiều cao cảm biến $\Delta z = 1.73\text{ m}$ (chuẩn KITTI), mô hình nhận diện được **13 hộp** gồm cả xe và người/vật thể. Chiều cao trung bình các hộp là $1.034\text{ m}$, các hộp bám khớp với cụm điểm trên mặt đường trong ảnh Side. Đây là baseline chuẩn. |
| **C** | 1.73 m | 0.32 m | 6 | 1.091 m | `boxes-demo-delta-1.73-voxel-0.32.json`<br>`side-demo-delta-1.73-voxel-0.32.png`<br>`summary.csv` | Giữ nguyên $\Delta z = 1.73\text{ m}$ nhưng tăng kích thước pillar gấp đôi ($0.16 \rightarrow 0.32\text{ m}$), số hộp phát hiện giảm mạnh từ **13 xuống còn 6 hộp**. Lưới pillar thô hơn làm gộp điểm và giảm đặc trưng không gian của các đối tượng nhỏ hoặc điểm thưa. |

### Trả lời câu hỏi phân tích:

* **A/B: Thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  * **Khác hoàn toàn.** Thay đổi $\Delta z$ ở đầu vào trước model làm thay đổi tọa độ z của toàn bộ đám mây điểm khi đưa vào biểu diễn voxel/pillar. Mạng nơ-ron nhận diện đối tượng dựa trên tương quan hình học 3D trong không gian đặc trưng. Ở lượt A, vì không bù $\Delta z$, toàn bộ đám mây điểm bị lệch khỏi khoảng cao độ học được của model, dẫn đến mạng chỉ tìm được **1 hộp** thay vì 13 hộp. Nếu chỉ dịch output sau inference bằng một hằng số thì số lượng hộp vẫn giữ nguyên là 13 hộp nhưng chỉ thay đổi vị trí $z$; còn đổi input trước inference làm mạng nhận diện lại và thay đổi cả số lượng hộp, phân loại lớp và độ tự tin (score).

* **B/C: Thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  * Khi tăng kích thước pillar từ $0.16\text{ m}$ lên $0.32\text{ m}$, số hộp giảm từ 13 xuống còn 6 hộp (giảm hơn 50%).
  * **Chưa đủ bằng chứng để khẳng định cấu hình nào tốt hơn tuyệt đối.** Số lượng hộp nhiều hơn ở lượt B chưa chứng minh tất cả 13 hộp đều đúng (có thể có false positive); ngược lại 6 hộp ở lượt C có thể lọc bớt nhiễu hoặc bỏ sót đối tượng thật (false negative). Cần đối chiếu với nhãn chuẩn (ground truth reference) và ảnh camera/nhiều góc nhìn mới đánh giá được độ chính xác (Precision/Recall).

* **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  * ROI chỉ xét cửa sổ phía trước (`front-window`: $x \in [0, 69.12]\text{ m}$), do đó các đối tượng phía sau hoặc ngoài biên quét không xuất hiện trong output — đây là do giới hạn ROI chứ không phải model bỏ sót.
  * Ảnh chiếu bên (Side view: trục $x-z$) chiếu toàn bộ các vật thể lên một mặt phẳng 2D, khiến các xe đỗ song song hoặc khác tọa độ $y$ bị chồng lấn lên nhau, đồng thời không thể xác định được góc quay quanh trục thẳng đứng (yaw) hay phân biệt đầu/đuôi xe. Vì vậy Side view chỉ dùng để kiểm tra cao độ $z$ và mặt đường, không dùng độc lập để kết luận hình học 3D.

* **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  * Cả 3 file JSON A, B, C đều là **kết quả dự đoán thô (raw predictions)** của mô hình pretrained KITTI trên tập demo, chưa qua phân xử (unadjudicated) và không thể import trực tiếp vào CVAT làm nhãn đúng.
  * Cần kiểm tra: đối chiếu 4 góc nhìn (Top, Side, Front, Xoay tự do) với đám mây điểm thật, kiểm tra ảnh camera cùng frame để xác thực class, hướng yaw đầu xe và loại bỏ các hộp dự đoán thừa/thiếu.

---

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | :---: | :---: | :---: | :--- | :--- |
| `case-correct` | **0 / 13** | 0 m | Không đổi | **Không phải lỗi pipeline** | Giữ nguyên phép chuyển hệ tọa độ nguồn; cả 13 hộp bám khít cụm điểm và mặt đường địa phương trong ảnh `side-correct.png`. |
| `case-batch-z` | **13 / 13** (100%) | **-1.805 m** | Không đổi | **DỪNG BATCH, KIỂM TRA TOÀN BỘ PIPELINE** | Tất cả 13/13 hộp đều bị trừ cùng một lượng $z_{ground} + \Delta z = 0.075 + 1.73 = 1.805\text{ m}$. Toàn bộ hộp chìm sâu xuống dưới mặt đường trong ảnh `side-batch-z.png`. Đây là lỗi quên phép cộng z ngược ở pipeline, tuyệt đối không sửa tay từng hộp. |
| `case-one-box-z` | **1 / 13** | **-1.805 m** | Không đổi | **Kiểm tra từng hộp / đối tượng** | Chỉ duy nhất hộp đầu tiên bị lệch $z$ xuống dưới $1.805\text{ m}$, 12 hộp còn lại vẫn ở vị trí chuẩn. Đây là lỗi đối tượng cục bộ; cần dùng nhiều góc nhìn để điều chỉnh lại hộp bị lỗi, không dừng cả batch. |

*Lưu ý*: Script helper `pipeline-qc-cases.py` tạo các biến đổi có chủ đích từ prediction lượt B nhằm phục vụ huấn luyện nhận diện lỗi, không phải kết quả detector riêng biệt và không phải ground truth.

---

## Nhận xét cá nhân

### 1. Nguyễn Đình Viễn (MSSV: 02148)
* **Vai trò**: Vận hành lệnh (Lượt A), Kiểm cấu hình/JSON (Lượt B), Xem hình học Side (Lượt C).
* **Quan sát A/B/C**: Ở lượt A khi gán $\Delta z = 0\text{ m}$, file `boxes-demo-delta-0-voxel-0.16.json` chỉ ghi nhận 1 hộp xe duy nhất. Nhưng sang lượt B với $\Delta z = 1.73\text{ m}$, số hộp tăng vọt lên 13. Điều này chứng minh việc đưa đúng độ cao cảm biến trước khi trích xuất đặc trưng là điều kiện tiên quyết để mô hình PointPillars hoạt động.
* **Diễn giải phép biến đổi z**: Công thức $z_{model} = z_{source} - z_{ground} - \Delta z$ chuẩn hóa đám mây điểm về mặt phẳng tham chiếu của mô hình; sau khi inference xong bắt buộc phải thực hiện phép biến đổi ngược $z_{source} = z_{model} + z_{ground} + \Delta z$ để đưa cuboid về đúng hệ tọa độ nguồn.
* **Quyết định lỗi batch**: Trong ca `case-batch-z`, khi thấy toàn bộ 13 hộp cùng tụt $1.805\text{ m}$, tôi quyết định dừng chỉnh sửa thủ công và yêu cầu kiểm tra pipeline chuyển đổi z.
* **Điều chưa chắc**: Chưa có ảnh camera đồng bộ cho frame demo để kiểm tra hướng quay yaw của xe ở khoảng cách xa $x > 40\text{ m}$.

### 2. Lê Đức Minh Quân (MSSV: 02126)
* **Vai trò**: Kiểm cấu hình/JSON (Lượt A), Xem hình học Side (Lượt B), Ghi log/Thư ký (Lượt C); trực tiếp thao tác chạy runner gói bundle.
* **Quan sát A/B/C**: Trong file `smoke.json`, cả 3 lượt đều passed với thời gian inference CPU từ 7–13 giây. Xem ảnh `side-demo-delta-1.73-voxel-0.16.png` ở lượt B thấy các bounding box bao trọn cụm điểm thân xe và đáy hộp tiếp xúc sát mặt đường tại $z \approx 0$.
* **Diễn giải phép biến đổi z**: Phép dịch z trước model tác động trực tiếp vào quá trình voxel hóa điểm, khác với việc tịnh tiến hình học đơn thuần sau khi đã có hộp.
* **Quyết định lỗi batch**: Khi 100% số hộp bị sai cùng một khoảng dịch z, nguyên nhân chắc chắn nằm ở code chuyển đổi hệ trục. Cần vá code chứ không thể gán nhãn thủ công bù lại.
* **Điều chưa chắc**: Một số cụm điểm thưa ở lượt B có score dao động từ 0.30–0.35, chưa thể khẳng định là đối tượng thật hay nhiễu nếu thiếu góc nhìn Front và Top.

### 3. Nguyễn Đức Hà (MSSV: 02105)
* **Vai trò**: Xem hình học Side (Lượt A), Ghi log/Thư ký (Lượt B), Vận hành lệnh (Lượt C).
* **Quan sát A/B/C**: Khi chuyển từ lượt B sang lượt C, kích thước pillar XY tăng từ $0.16\text{ m}$ lên $0.32\text{ m}$, số lượng hộp phát hiện giảm từ 13 xuống còn 6 hộp (theo `summary.csv`). Các hộp bị mất chủ yếu là các đối tượng nhỏ có ít điểm LiDAR.
* **Diễn giải phép biến đổi z**: Trục z trong hệ KITTI có cảm biến nằm cách mặt đất khoảng $1.73\text{ m}$. Bỏ qua tham số này sẽ khiến mạng hiểu lầm mặt đường nằm quá cao hoặc quá thấp, làm hỏng toàn bộ feature map 2D giả lập (pseudo-image).
* **Quyết định lỗi batch**: Ca `case-one-box-z` chỉ có 1 hộp bị chìm, các hộp còn lại bình thường. Tôi quyết định kiểm tra riêng hộp đó trên giao diện 3D và điều chỉnh đáy hộp bám mặt đường cục bộ, giữ nguyên các hộp khác.
* **Điều chưa chắc**: Chưa rõ với các xe bị che khuất một nửa thân thì thuật toán ước lượng kích thước hộp theo cụm điểm nhìn thấy hay theo kích thước chuẩn trung bình của taxonomy.

### 4. Bùi Phương Nam (MSSV: 02134)
* **Vai trò**: Ghi log/Thư ký (Lượt A), Vận hành lệnh (Lượt B), Kiểm cấu hình/JSON (Lượt C).
* **Quan sát A/B/C**: Giá trị `mean_z` ở lượt A là $0.330\text{ m}$, trong khi ở lượt B là $1.034\text{ m}$ và lượt C là $1.091\text{ m}$. Sự chênh lệch $mean\_z$ giữa A và B phản ánh rõ rệt việc bù đắp độ cao sensor.
* **Diễn giải phép biến đổi z**: Hiểu rõ hai chiều biến đổi: chiều thuận chuẩn hóa dữ liệu vào tensor mạng, chiều ngược khôi phục tọa độ vật lý cho việc hiển thị và đánh giá trên CVAT.
* **Quyết định lỗi batch**: Quyết định phân biệt rạch ròi giữa lỗi hệ thống (toàn batch cùng sai $\Delta z$) và lỗi ngẫu nhiên (chỉ 1 hộp sai do điểm thưa/nhiễu). Lỗi hệ thống phải sửa ở tầng phần mềm.
* **Điều chưa chắc**: Kênh RGB=0 hằng số thay thế cho intensity thực có thể làm giảm khả năng nhận diện các vật liệu phản xạ mạnh như biển báo hoặc đèn xe.

---

## LC ghi nhận riêng
# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm 5 — Nhóm siêu nhân / Phòng Lab Day 13
- Thành viên: xem `TEAMMATES.md` (Nguyễn Đình Viễn - 02148, Lê Đức Minh Quân - 02126, Nguyễn Đức Hà - 02105, Bùi Phương Nam - 02134; vai trò xoay vòng từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Đức Minh Quân (cùng cả nhóm vận hành); 2026-10-01; Linux x86_64 / amd64 (Docker Desktop).
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - Frame: `demo.pcd` (mẫu KITTI 000008 chuyển đổi theo CC BY-NC-SA 3.0)
  - Input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Chạy local offline theo hợp đồng runner `--network none`.
- Checkpoint: PointPillars KITTI có sẵn trong image:
  - Path: `/opt/PointPillars/pretrained/epoch_160.pth`
  - Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (`range: 0.0, -39.68, -3.0, 69.12, 39.68, 1.0`); score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - PCD KITTI Student lược bỏ reflectance thực, sử dụng RGB=0 làm placeholder (kênh hằng số cho adapter).
  - $z_{ground}$ được ước lượng từ đám mây điểm: $z_{ground} = 0.075\text{ m}$.

---

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| **A** | 0 m | 0.16 m | 1 | 0.330 m | `boxes-demo-delta-0-voxel-0.16.json`<br>`side-demo-delta-0-voxel-0.16.png`<br>`summary.csv` | Khi $\Delta z = 0\text{ m}$, mô hình chỉ phát hiện duy nhất **1 hộp** (`vehicles`, score 0.322) ở vị trí $x \approx 13.15, y \approx -0.45, z \approx 0.33$. Đám mây điểm không được bù chiều cao cảm biến nên nằm ngoài phân phối độ cao thông thường của KITTI, dẫn tới bỏ sót 12 đối tượng còn lại. |
| **B** | 1.73 m | 0.16 m | 13 | 1.034 m | `boxes-demo-delta-1.73-voxel-0.16.json`<br>`side-demo-delta-1.73-voxel-0.16.png`<br>`summary.csv` | Khi bù chiều cao cảm biến $\Delta z = 1.73\text{ m}$ (chuẩn KITTI), mô hình nhận diện được **13 hộp** gồm cả xe và người/vật thể. Chiều cao trung bình các hộp là $1.034\text{ m}$, các hộp bám khớp với cụm điểm trên mặt đường trong ảnh Side. Đây là baseline chuẩn. |
| **C** | 1.73 m | 0.32 m | 6 | 1.091 m | `boxes-demo-delta-1.73-voxel-0.32.json`<br>`side-demo-delta-1.73-voxel-0.32.png`<br>`summary.csv` | Giữ nguyên $\Delta z = 1.73\text{ m}$ nhưng tăng kích thước pillar gấp đôi ($0.16 \rightarrow 0.32\text{ m}$), số hộp phát hiện giảm mạnh từ **13 xuống còn 6 hộp**. Lưới pillar thô hơn làm gộp điểm và giảm đặc trưng không gian của các đối tượng nhỏ hoặc điểm thưa. |

### Trả lời câu hỏi phân tích:

* **A/B: Thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  * **Khác hoàn toàn.** Thay đổi $\Delta z$ ở đầu vào trước model làm thay đổi tọa độ z của toàn bộ đám mây điểm khi đưa vào biểu diễn voxel/pillar. Mạng nơ-ron nhận diện đối tượng dựa trên tương quan hình học 3D trong không gian đặc trưng. Ở lượt A, vì không bù $\Delta z$, toàn bộ đám mây điểm bị lệch khỏi khoảng cao độ học được của model, dẫn đến mạng chỉ tìm được **1 hộp** thay vì 13 hộp. Nếu chỉ dịch output sau inference bằng một hằng số thì số lượng hộp vẫn giữ nguyên là 13 hộp nhưng chỉ thay đổi vị trí $z$; còn đổi input trước inference làm mạng nhận diện lại và thay đổi cả số lượng hộp, phân loại lớp và độ tự tin (score).

* **B/C: Thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  * Khi tăng kích thước pillar từ $0.16\text{ m}$ lên $0.32\text{ m}$, số hộp giảm từ 13 xuống còn 6 hộp (giảm hơn 50%).
  * **Chưa đủ bằng chứng để khẳng định cấu hình nào tốt hơn tuyệt đối.** Số lượng hộp nhiều hơn ở lượt B chưa chứng minh tất cả 13 hộp đều đúng (có thể có false positive); ngược lại 6 hộp ở lượt C có thể lọc bớt nhiễu hoặc bỏ sót đối tượng thật (false negative). Cần đối chiếu với nhãn chuẩn (ground truth reference) và ảnh camera/nhiều góc nhìn mới đánh giá được độ chính xác (Precision/Recall).

* **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  * ROI chỉ xét cửa sổ phía trước (`front-window`: $x \in [0, 69.12]\text{ m}$), do đó các đối tượng phía sau hoặc ngoài biên quét không xuất hiện trong output — đây là do giới hạn ROI chứ không phải model bỏ sót.
  * Ảnh chiếu bên (Side view: trục $x-z$) chiếu toàn bộ các vật thể lên một mặt phẳng 2D, khiến các xe đỗ song song hoặc khác tọa độ $y$ bị chồng lấn lên nhau, đồng thời không thể xác định được góc quay quanh trục thẳng đứng (yaw) hay phân biệt đầu/đuôi xe. Vì vậy Side view chỉ dùng để kiểm tra cao độ $z$ và mặt đường, không dùng độc lập để kết luận hình học 3D.

* **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  * Cả 3 file JSON A, B, C đều là **kết quả dự đoán thô (raw predictions)** của mô hình pretrained KITTI trên tập demo, chưa qua phân xử (unadjudicated) và không thể import trực tiếp vào CVAT làm nhãn đúng.
  * Cần kiểm tra: đối chiếu 4 góc nhìn (Top, Side, Front, Xoay tự do) với đám mây điểm thật, kiểm tra ảnh camera cùng frame để xác thực class, hướng yaw đầu xe và loại bỏ các hộp dự đoán thừa/thiếu.

---

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | :---: | :---: | :---: | :--- | :--- |
| `case-correct` | **0 / 13** | 0 m | Không đổi | **Không phải lỗi pipeline** | Giữ nguyên phép chuyển hệ tọa độ nguồn; cả 13 hộp bám khít cụm điểm và mặt đường địa phương trong ảnh `side-correct.png`. |
| `case-batch-z` | **13 / 13** (100%) | **-1.805 m** | Không đổi | **DỪNG BATCH, KIỂM TRA TOÀN BỘ PIPELINE** | Tất cả 13/13 hộp đều bị trừ cùng một lượng $z_{ground} + \Delta z = 0.075 + 1.73 = 1.805\text{ m}$. Toàn bộ hộp chìm sâu xuống dưới mặt đường trong ảnh `side-batch-z.png`. Đây là lỗi quên phép cộng z ngược ở pipeline, tuyệt đối không sửa tay từng hộp. |
| `case-one-box-z` | **1 / 13** | **-1.805 m** | Không đổi | **Kiểm tra từng hộp / đối tượng** | Chỉ duy nhất hộp đầu tiên bị lệch $z$ xuống dưới $1.805\text{ m}$, 12 hộp còn lại vẫn ở vị trí chuẩn. Đây là lỗi đối tượng cục bộ; cần dùng nhiều góc nhìn để điều chỉnh lại hộp bị lỗi, không dừng cả batch. |

*Lưu ý*: Script helper `pipeline-qc-cases.py` tạo các biến đổi có chủ đích từ prediction lượt B nhằm phục vụ huấn luyện nhận diện lỗi, không phải kết quả detector riêng biệt và không phải ground truth.

---

## Nhận xét cá nhân

### 1. Nguyễn Đình Viễn (MSSV: 02148)
* **Vai trò**: Vận hành lệnh (Lượt A), Kiểm cấu hình/JSON (Lượt B), Xem hình học Side (Lượt C).
* **Quan sát A/B/C**: Ở lượt A khi gán $\Delta z = 0\text{ m}$, file `boxes-demo-delta-0-voxel-0.16.json` chỉ ghi nhận 1 hộp xe duy nhất. Nhưng sang lượt B với $\Delta z = 1.73\text{ m}$, số hộp tăng vọt lên 13. Điều này chứng minh việc đưa đúng độ cao cảm biến trước khi trích xuất đặc trưng là điều kiện tiên quyết để mô hình PointPillars hoạt động.
* **Diễn giải phép biến đổi z**: Công thức $z_{model} = z_{source} - z_{ground} - \Delta z$ chuẩn hóa đám mây điểm về mặt phẳng tham chiếu của mô hình; sau khi inference xong bắt buộc phải thực hiện phép biến đổi ngược $z_{source} = z_{model} + z_{ground} + \Delta z$ để đưa cuboid về đúng hệ tọa độ nguồn.
* **Quyết định lỗi batch**: Trong ca `case-batch-z`, khi thấy toàn bộ 13 hộp cùng tụt $1.805\text{ m}$, tôi quyết định dừng chỉnh sửa thủ công và yêu cầu kiểm tra pipeline chuyển đổi z.
* **Điều chưa chắc**: Chưa có ảnh camera đồng bộ cho frame demo để kiểm tra hướng quay yaw của xe ở khoảng cách xa $x > 40\text{ m}$.

### 2. Lê Đức Minh Quân (MSSV: 02126)
* **Vai trò**: Kiểm cấu hình/JSON (Lượt A), Xem hình học Side (Lượt B), Ghi log/Thư ký (Lượt C); trực tiếp thao tác chạy runner gói bundle.
* **Quan sát A/B/C**: Trong file `smoke.json`, cả 3 lượt đều passed với thời gian inference CPU từ 7–13 giây. Xem ảnh `side-demo-delta-1.73-voxel-0.16.png` ở lượt B thấy các bounding box bao trọn cụm điểm thân xe và đáy hộp tiếp xúc sát mặt đường tại $z \approx 0$.
* **Diễn giải phép biến đổi z**: Phép dịch z trước model tác động trực tiếp vào quá trình voxel hóa điểm, khác với việc tịnh tiến hình học đơn thuần sau khi đã có hộp.
* **Quyết định lỗi batch**: Khi 100% số hộp bị sai cùng một khoảng dịch z, nguyên nhân chắc chắn nằm ở code chuyển đổi hệ trục. Cần vá code chứ không thể gán nhãn thủ công bù lại.
* **Điều chưa chắc**: Một số cụm điểm thưa ở lượt B có score dao động từ 0.30–0.35, chưa thể khẳng định là đối tượng thật hay nhiễu nếu thiếu góc nhìn Front và Top.

### 3. Nguyễn Đức Hà (MSSV: 02105)
* **Vai trò**: Xem hình học Side (Lượt A), Ghi log/Thư ký (Lượt B), Vận hành lệnh (Lượt C).
* **Quan sát A/B/C**: Khi chuyển từ lượt B sang lượt C, kích thước pillar XY tăng từ $0.16\text{ m}$ lên $0.32\text{ m}$, số lượng hộp phát hiện giảm từ 13 xuống còn 6 hộp (theo `summary.csv`). Các hộp bị mất chủ yếu là các đối tượng nhỏ có ít điểm LiDAR.
* **Diễn giải phép biến đổi z**: Trục z trong hệ KITTI có cảm biến nằm cách mặt đất khoảng $1.73\text{ m}$. Bỏ qua tham số này sẽ khiến mạng hiểu lầm mặt đường nằm quá cao hoặc quá thấp, làm hỏng toàn bộ feature map 2D giả lập (pseudo-image).
* **Quyết định lỗi batch**: Ca `case-one-box-z` chỉ có 1 hộp bị chìm, các hộp còn lại bình thường. Tôi quyết định kiểm tra riêng hộp đó trên giao diện 3D và điều chỉnh đáy hộp bám mặt đường cục bộ, giữ nguyên các hộp khác.
* **Điều chưa chắc**: Chưa rõ với các xe bị che khuất một nửa thân thì thuật toán ước lượng kích thước hộp theo cụm điểm nhìn thấy hay theo kích thước chuẩn trung bình của taxonomy.

### 4. Bùi Phương Nam (MSSV: 02134)
* **Vai trò**: Ghi log/Thư ký (Lượt A), Vận hành lệnh (Lượt B), Kiểm cấu hình/JSON (Lượt C).
* **Quan sát A/B/C**: Giá trị `mean_z` ở lượt A là $0.330\text{ m}$, trong khi ở lượt B là $1.034\text{ m}$ và lượt C là $1.091\text{ m}$. Sự chênh lệch $mean\_z$ giữa A và B phản ánh rõ rệt việc bù đắp độ cao sensor.
* **Diễn giải phép biến đổi z**: Hiểu rõ hai chiều biến đổi: chiều thuận chuẩn hóa dữ liệu vào tensor mạng, chiều ngược khôi phục tọa độ vật lý cho việc hiển thị và đánh giá trên CVAT.
* **Quyết định lỗi batch**: Quyết định phân biệt rạch ròi giữa lỗi hệ thống (toàn batch cùng sai $\Delta z$) và lỗi ngẫu nhiên (chỉ 1 hộp sai do điểm thưa/nhiễu). Lỗi hệ thống phải sửa ở tầng phần mềm.
* **Điều chưa chắc**: Kênh RGB=0 hằng số thay thế cho intensity thực có thể làm giảm khả năng nhận diện các vật liệu phản xạ mạnh như biển báo hoặc đèn xe.

---

## LC ghi nhận riêng

> LC ghi nhận ngày 01/10/2026. **Kết luận: ĐẠT.**

- **Quyền dùng PCD/image và đúng ca:** Gói Student KITTI 000008 (giấy phép CC BY-NC-SA 3.0), không dùng dữ liệu Robotaxi. Input SHA-256 `3b5ea3da…` và image `sha256:e03983bd…` (amd64) khớp `smoke.json`.
- **Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:** Chạy thật (`executed-by-group`). `smoke.json` passed (không đổi): nạp image 78,7 s, A/B/C 13,2 / 8,9 / 7,7 s, kết quả 1/13/6.
- **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** Đủ `run-A/B/C` và `qc-cases`. Không sửa JSON. Không đưa ca lỗi vào CVAT.
- **Nhận xét từng thành viên và quyết định dừng pipeline:** Đủ 4 người, phép z đúng, tính đúng 0,075 + 1,73 = 1,805, quyết định batch/one-box đúng. Nội dung giữ nguyên: lượt C vẫn ghi "mất đối tượng nhỏ" (thực tế C mất hết xe, 6/6 là `pedestrian`) và A "bỏ sót 12 đối tượng" (coi B là đáp án).
- **Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:** **Đồng ý chuyển sang chỉnh/QC.**

**Nên sửa:**
1. Sửa nhận xét B/C: mở JSON lượt C, cột `label` cho thấy 6/6 `pedestrian`, không còn `vehicles` (chưa sửa ở lần gửi lại).
2. Bỏ ý "A bỏ sót 12 đối tượng": chưa có nhãn đúng thì chưa nói được A bỏ sót.
3. Chuyển họ tên, MSSV từ báo cáo sang `TEAMMATES.md`.


