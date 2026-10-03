# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân — Nguyễn Minh Trung
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy: Nguyễn Minh Trung; ngày/giờ: 2026-10-02 23:45 (GMT+7); hệ máy/architecture: Windows amd64
- Image tag và image ID: `day13-pointpillars:lc-20261001-amd64` / `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
- PCD được cấp / frame_id: `demo` (PCD KITTI Student trong gói student-prelabel-amd64); input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI pretrained `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity: RGB=0, adapter kênh hằng (PCD KITTI Student đã bỏ reflectance thật). z_ground ước lượng từ PCD: 0.075 m

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ phát hiện 1 hộp class vehicles. Model với delta=0 (không dịch z trước inference) gần như không nhận diện được các vật thể trong cảnh này. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Phát hiện 13 hộp: vehicles=10, pedestrian=2, two-wheels=1. Dịch z trước inference (delta=1.73) giúp model nhận diện được nhiều vật thể hơn rất nhiều. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Chỉ phát hiện 6 hộp, toàn bộ là pedestrian=6. Khi tăng kích thước pillar lên 0.32 m, model bỏ sót hoàn toàn vehicles và two-wheels; có thể do điểm bị gom vào pillar lớn hơn làm mất đặc trưng kích thước nhỏ. |

- A/B — chỉ đổi delta: A có **1** hộp; B có **13** hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` (lượt A) gần như trống, trong khi `side-demo-delta-1.73-voxel-0.16.png` (lượt B) thể hiện nhiều cụm hộp ở vùng phía trước. Đây là chạy lại model trên input đã dịch z khác, không chỉ dịch hộp cũ — vì model đã xử lý phân phối điểm khác nhau nên số hộp và class đều thay đổi. Điều còn chưa chắc: không rõ tại sao delta=0 chỉ cho 1 hộp mà không phải 0 hộp — có thể mặt đường còn một cụm điểm nằm đúng trong ROI.
- B/C — chỉ đổi pillar: B có **13** hộp (vehicles, pedestrian, two-wheels); C có **6** hộp (chỉ pedestrian). Ảnh `side-demo-delta-1.73-voxel-0.32.png` (lượt C) thể hiện ít hộp hơn hẳn và thiếu vùng vehicles. Khi pillar XY tăng từ 0.16→0.32, các điểm của xe hơi bị gom vào ô lớn hơn, làm mất đặc trưng shape nên model không còn phân biệt được class vehicles và two-wheels. Không đủ bằng chứng để kết luận cấu hình nào "tốt hơn" vì không có ground truth.
- Giới hạn ROI và góc Side: Ảnh Side chiếu x-z, nên các vật thể bị chồng theo trục y. Không thể phân biệt chính xác yaw hoặc vị trí y từ ảnh Side; chỉ dùng làm điểm bắt đầu kiểm tra, chưa đủ để duyệt từng hộp.
- JSON nào còn chưa đủ cơ sở để import: Cả ba lượt đều là kết quả trên PCD KITTI demo không có intensity thật, không import vào CVAT Robotaxi. Cần kiểm: đối chiếu từng hộp với ảnh camera thật và góc nhìn đa chiều trước khi xác nhận.

## Ca QC có kiểm soát — không import CVAT

Helper tạo ba ca biến đổi có chủ đích từ prediction thật của lượt B (13 hộp). Đây không phải kết quả inference riêng và không phải nhãn đúng.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch trung bình | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 (không lệch) | Không đổi | Không cần dừng — đây là prediction gốc từ B, không có lỗi z được áp vào | `qc-cases/case-correct.json`: z các hộp nằm trong khoảng 0.70–1.43 m, phân phối hợp lý gần mặt đường |
| case-batch-z | 13 / 13 | ~-1.81 m (toàn bộ hộp bị trừ z_ground+delta = 0.075+1.73) | Không đổi | **Dừng batch và báo LC kiểm transform** — cả 13 hộp đều lệch cùng một lượng, x/y/yaw không thay đổi, đây là lỗi chuyển hệ tọa độ toàn batch | `qc-cases/case-batch-z.json`: z các hộp âm (~-0.38 đến -1.11 m), tức là tất cả hộp nằm dưới mặt đường — không thể là lỗi từng đối tượng |
| case-one-box-z | 1 / 13 | ~-1.81 m (chỉ hộp đầu tiên bị trừ) | Không đổi | **Kiểm thêm hình học từng đối tượng** — chỉ 1 hộp lệch, 12 hộp còn lại bình thường, đây là lỗi cục bộ một đối tượng, không phải lỗi pipeline | `qc-cases/case-one-box-z.json`: hộp đầu tiên (vehicles, x≈8.09) có z=-0.88 m trong khi các hộp khác vẫn z>0.7 m |

Ghi rõ: helper tạo biến đổi có chủ đích từ prediction thật của B, không phải kết quả inference riêng hoặc nhãn đúng. Không import các ca này vào CVAT.

## Nhận xét cá nhân

**Nguyễn Minh Trung — MSSV: 2A2020602191**

- **Vai trò đã làm:** Thực hiện toàn bộ — vận hành lệnh, đọc JSON/CSV, quan sát ảnh Side và phân tích ba ca QC.
- **Quan sát A/B/C:** Trong file `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, hộp đầu tiên (vehicles, x≈8.09, z≈0.92) xuất hiện rõ trên ảnh `side-demo-delta-1.73-voxel-0.16.png` ở vùng x=8–9 m, z≈0.9 m. Cùng vùng này trên ảnh `side-demo-delta-0-voxel-0.16.png` (lượt A) không thấy hộp nào — điều này xác nhận phép dịch z trước inference thay đổi đầu vào model, không chỉ dịch hộp đầu ra.
- **Diễn giải phép z thuận/ngược:** Phép thuận: `z_model = z_source - z_ground - delta` (dịch điểm PCD trước khi vào model). Phép ngược: `z_source = z_model + z_ground + delta` (chuyển hộp model về hệ nguồn để xuất JSON). JSON đã áp phép ngược sẵn, không cần cộng thêm delta khi đọc. Nếu phép ngược bị bỏ sót, tất cả hộp sẽ nằm ở vùng z âm — đúng với những gì thấy trong case-batch-z.
- **Quyết định lỗi batch:** Với case-batch-z, 13/13 hộp đều có z âm và cùng lệch ~1.81 m, trong khi x/y/yaw không thay đổi. Đây là dấu hiệu rõ ràng của lỗi pipeline (bỏ sót phép chuyển hệ ngược), nên hành động là **dừng sửa tay và báo LC kiểm transform**, không chỉnh từng hộp.
- **Điều chưa chắc:** Chưa rõ tại sao lượt A (delta=0) vẫn cho 1 hộp thay vì 0 — cần kiểm thêm xem hộp đó có thực sự khớp với vật thể nào trong ảnh camera hay là false positive. Ngoài ra, lượt C bỏ sót toàn bộ vehicles có thể do pillar lớn hơn hoặc do đặc điểm PCD demo thiếu intensity; chưa thể kết luận pillar 0.32 luôn kém hơn.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
