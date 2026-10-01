# K4 – Day 13: Robotaxi LiDAR 3D Object

## Giới thiệu

Đây là repository phục vụ bài thực hành **Day 13 – Robotaxi A: LiDAR 3D Object** trong chương trình AI20K – VinUni.

Bài học tập trung vào quy trình:

- Chạy và đối chiếu PointPillars trên dữ liệu PCD KITTI minh họa.
- So sánh ba cấu hình **A / B / C**.
- Phân tích ảnh hưởng của `delta` và kích thước pillar XY.
- Kiểm tra phép chuyển đổi trục `z` và phân biệt lỗi pipeline với lỗi của một cuboid.
- Nạp pre-label vào job Robotaxi và kiểm chứng từng cuboid.
- Sửa cuboid 3D bằng nhiều góc nhìn và ảnh camera.
- Thực hiện vòng **v1 → QC feedback → v2**.
- Viết báo cáo thí nghiệm và nhận xét QC có bằng chứng.

> **Lưu ý:** Repository này chỉ chứa tài liệu Markdown phục vụ nộp bài. Dữ liệu Robotaxi, PCD, ảnh camera và annotation được giữ trong hệ thống được cấp theo quy định của bài học.

---

## Cấu trúc repository

```text
K4-DAY13-HoVaTen-MSSV/
├── README.md
└── report/
    └── K4-DAY13-TenNhom/
        ├── TEAMMATES.md
        └── PRE-LABEL-REPORT.md
```

### Các file

| File | Nội dung |
|---|---|
| `README.md` | Giới thiệu repository và phạm vi bài thực hành |
| `TEAMMATES.md` | Danh sách thành viên, MSSV và vai trò |
| `PRE-LABEL-REPORT.md` | Báo cáo PointPillars, A/B/C, phép đổi z, QC cases và nhận xét của từng thành viên |

---

## Nội dung thực hành

### 1. PointPillars – A/B/C

Ba lượt chạy sử dụng cùng PCD và checkpoint, chỉ thay đổi cấu hình theo bài:

| Lượt | `delta` | Pillar XY |
|---|---:|---:|
| A | `0` | `0.16 m` |
| B | `1.73 m` | `0.16 m` |
| C | `1.73 m` | `0.32 m` |

Trong đó:

- **A/B** dùng để quan sát ảnh hưởng của phép dịch `z` trước inference.
- **B/C** dùng để quan sát ảnh hưởng của kích thước pillar XY.
- B được sử dụng làm mốc đối chiếu và tạo các ca QC.

Kết quả được đối chiếu từ:

- `summary.csv`
- `boxes-*.json`
- ảnh `side-*.png`
- `smoke.json`
- các file trong `qc-cases/`

Các kết luận trong báo cáo dựa trên output thực tế của nhóm.

---

## 2. Kiểm tra phép chuyển đổi z

Pipeline sử dụng quan hệ:

```text
z_model  = z_source - z_ground - delta
z_source = z_model  + z_ground + delta
```

Báo cáo phân biệt:

- lỗi ảnh hưởng nhiều hộp hoặc cả batch → cần kiểm tra nguyên nhân pipeline;
- lỗi chỉ xuất hiện ở một cuboid → kiểm tra riêng class, vị trí, kích thước, hướng hoặc đáy hộp;
- trường hợp chưa đủ bằng chứng → ghi rõ là chưa chắc thay vì tự suy đoán.

Không tự cộng/trừ `delta` thêm lần nữa khi đọc JSON đã được runner xuất về hệ tọa độ nguồn.

---

## 3. Annotation Robotaxi

Phần cá nhân được thực hiện trên job Robotaxi được cấp qua portal.

Quy trình:

1. Nhận đúng job nguồn.
2. Nạp pre-label đúng frame.
3. Kiểm tra từng cuboid.
4. Kiểm tra class, tâm, kích thước, hướng và đáy.
5. Đối chiếu các góc **Trên / Trước / Bên** và ảnh camera.
6. Save annotation.
7. Nộp snapshot **v1** vào hàng đợi QC.
8. Nhận feedback.
9. Sửa nguồn theo feedback.
10. Save và nộp **v2 / phản hồi** trên portal.

Pre-label chỉ là prediction ban đầu. Mỗi cuboid vẫn phải được kiểm chứng.

---

## 4. QC Feedback

Một feedback cần giúp người khác tìm được lỗi và biết cần kiểm tra gì.

Feedback nên gồm:

- **Loại lỗi**
- **ID QC hoặc vùng cần tìm nếu là đối tượng thiếu**
- **Bằng chứng / góc nhìn đã kiểm tra**
- **Đề xuất sửa hoặc lý do giữ nguyên**

Ví dụ về loại vấn đề:

- Sai class
- Thiếu đối tượng
- Hộp thừa
- Sai vị trí
- Sai kích thước
- Sai hướng
- Chưa chắc
- Không thấy lỗi trong phần đã rà

Feedback không chỉ ghi “hộp sai”; cần chỉ ra bằng chứng để người sửa có thể kiểm tra lại.

---

## 5. Phân quyền và dữ liệu

Repository này không chứa:

- PCD Robotaxi
- ảnh camera Robotaxi
- annotation dữ liệu thật
- dữ liệu hoặc artifact được yêu cầu giữ trong hệ thống được cấp

Các dữ liệu thực hành được giữ ở CVAT, portal hoặc thư mục được phép theo hướng dẫn của lớp.

---

## 6. Trạng thái bài nộp

### GitHub

Hai file Markdown bắt buộc:

```text
report/K4-DAY13-TenNhom/TEAMMATES.md
report/K4-DAY13-TenNhom/PRE-LABEL-REPORT.md
```

### CVAT / Portal

Các thao tác annotation được thực hiện trực tiếp trên hệ thống được cấp:

```text
Pre-label → Source Annotation → v1 → QC Feedback → v2
```

### VLearn

Nộp **link repository cá nhân** theo mục **Nộp bài** của Day 13.

---

## Nguồn bài học

- **Day 13 – Robotaxi A: LiDAR 3D Object**
- Repository bài học: `VinUni-AI20k/K4-L2L3-Day13-Robotaxi-LiDAR-3D-Object-Student`

---

## Thành viên

Xem `report/K4-DAY13-TenNhom/TEAMMATES.md` để biết danh sách thành viên và vai trò của từng người.

---

**K4 – AI20K · VinUni**
