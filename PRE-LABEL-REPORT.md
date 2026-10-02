# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Thực hành cá nhân / Phòng Lab Day 13
- Thành viên: Đoàn Văn Thắng — MSSV: 2A202602327 (xem `TEAMMATES.md`).
- Trạng thái: `executed-individually` (kết hợp đối chiếu `provided-results` từ bộ benchmark kiểm chứng chính thức của tài liệu `bundle/VALIDATION.md`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Đoàn Văn Thắng; 02/10/2026; Windows 11 x86_64 / Intel-AMD (Docker Linux Container CPU).
- Image tag và image ID; phiên bản repo: `pointpillars:kitti-cpu` (offline bundle), revision base `0831856`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (mẫu KITTI demo `000008`, 17.238 điểm, giấy phép CC BY-NC-SA 3.0, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`).
- Checkpoint: PointPillars KITTI có sẵn trong image (`/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window (`x: [0.0, 69.12]`, `y: [-39.68, 39.68]`, `z: [-3.0, 1.0]`); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - Dữ liệu `demo.pcd` đã loại bỏ reflectance thật (dùng placeholder uint32 rgb=0). Model sử dụng adapter kênh hằng với 2 lượt đọc: reflectance 0.0 để nhận diện lớp `vehicles`, và reflectance 0.7 để nhận diện lớp `pedestrian` và `two-wheels`.
  - Nguồn `z_ground`: Ước lượng mặt đất cục bộ từ đám mây điểm đầu vào trong phạm vi front-window.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | -0.124 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ nhận diện được đúng 1 hộp duy nhất ở cự ly gần ($x \approx 8$ m). Khi $\delta=0$, đám mây điểm không được hạ độ cao tương thích với chiều cao sensor KITTI ($\approx 1.73$ m), dẫn đến mạng trích xuất đặc trưng bị lệch phân bố z và bỏ sót gần như toàn bộ xe phía trước. |
| B | 1.73 | 0.16 | 13 | 0.382 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Nhận diện được 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels) rải đều dọc theo hành lang quan sát ($x \in [5, 45]$ m). Bounding box khớp tốt với cụm điểm xe trên hình chiếu Side. Đây là mốc so sánh chuẩn (baseline). |
| C | 1.73 | 0.32 | 6 | 0.514 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Số hộp giảm mạnh từ 13 xuống còn 6 hộp (toàn bộ 6 hộp đều là pedestrian). Khi tăng kích thước cạnh pillar lên 0.32 m (diện tích voxel gấp 4 lần), độ phân giải không gian bị thô ráp, làm mất hoàn toàn 10 xe hơi do đặc trưng hình học bị nhòe và trượt ngưỡng score 0.3. |

- **A/B — chỉ đổi delta:** A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` so với `side-demo-delta-1.73-voxel-0.16.png` khác rõ rệt ở dải khoảng cách $x \approx 10 - 45$ m: ở A không có hộp xe nào được dựng, trong khi ở B phát hiện đủ 10 xe bám sát mặt đất. Đây là chạy lại model trên input khác (phân bố cao độ $z$ trong các cột pillar thay đổi làm mạng trích xuất đặc trưng voxel hoàn toàn khác), không chỉ dịch hộp cũ; điều em còn chưa chắc là độ dốc mặt đường thực tế có thể làm $z_{ground}$ ước lượng bị sai số, ảnh hưởng đến delta hiệu dụng.
- **B/C — chỉ đổi pillar:** B có 13 hộp; C có 6 hộp. Ảnh `side-*.png` và file JSON cho thấy ở C mất hoàn toàn lớp `vehicles` (10 xe ở B biến mất, chỉ còn lại 6 hộp `pedestrian`). Kích thước pillar lớn 0.32 m làm gộp quá nhiều điểm dị biệt vào cùng một pillar, làm mờ ranh giới vật thể lớn. Có đủ bằng chứng để kết luận tốt hơn không? **Không đủ bằng chứng để kết luận cấu hình nào "tốt hơn" một cách tuyệt đối**, nhưng rõ ràng pillar 0.16 m phù hợp hơn với checkpoint pretrained hiện tại vốn được tối ưu ở voxel 0.16 m.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - Giới hạn ROI `front-window` ($x \in [0, 69.12]$ m) chỉ quét vùng không gian phía trước xe. Các vật thể ở hai bên sườn xa hoặc phía sau xe ($>90^\circ$ đến $180^\circ$) nằm ngoài vùng cắt dữ liệu, do đó việc không có hộp ở các vùng này là do thiết lập ROI, không phải do model bỏ sót (miss).
  - Góc nhìn `Side` là hình chiếu đứng $x-z$, cho phép kiểm tra trực quan chiều cao $z$, độ tiếp xúc mặt đất và chiều dài $x$. Tuy nhiên, góc này **hoàn toàn che khuất trục $y$ (bề rộng xe) và không thể đọc được góc xoay hướng (yaw)**. Muốn kiểm tra yaw bắt buộc phải kết hợp góc nhìn từ trên xuống (Bird's Eye View - BEV) hoặc 3D viewer.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - **Cả 3 file JSON A, B, C đều chưa đủ cơ sở để import vào hệ thống CVAT Robotaxi.** Lý do: Đây là kết quả inference trên tập dữ liệu demo KITTI với intensity giả lập (constant), chưa phản ánh đúng camera/LiDAR của xe Robotaxi thật và chưa qua quy trình kiểm duyệt nhãn chuẩn (reference annotation). Cần đối chiếu với ảnh camera đồng bộ và kiểm tra phân bố cụm điểm 3D trước khi sử dụng.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| **case-correct** | 0 / 13 | 0 m | Không | **Kiểm từng hộp (hoặc giữ nguyên)** | Bản sao nguyên vẹn từ prediction B (`training_only: true`). Tọa độ z bám đúng mặt đường, không có lỗi chuyển đổi cao độ. |
| **case-batch-z** | 13 / 13 (100%) | Lệch $-(\delta + z_{ground}) \approx -3.46$ m | Không | **DỪNG BATCH (Stop batch)** | 100% số hộp trong file đều bị chìm sâu xuống dưới mặt đất cùng một khoảng dịch cố định đúng bằng $\delta + z_{ground}$. Đây là lỗi hệ thống (systemic pipeline error) do script bỏ quên bước biến đổi ngược cao độ ($z_{source} = z_{model} + \delta + z_{ground}$). Cần dừng toàn bộ batch để sửa code xử lý, tuyệt đối không duyệt import. |
| **case-one-box-z** | 1 / 13 | Lệch $-(\delta + z_{ground})$ duy nhất ở hộp đầu tiên | Không | **KIỂM TỪNG HỘP (Inspect per-box)** | Chỉ có duy nhất 1 hộp đầu tiên bị tụt z, 12 hộp còn lại vẫn nằm đúng trên mặt đất. Đây là lỗi đối tượng cục bộ (outlier / false positive), pipeline chuyển đổi tọa độ không bị hỏng. Chỉ cần người kiểm duyệt chỉnh sửa hoặc loại bỏ riêng hộp lỗi đó trong CVAT. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Cá nhân thực hiện: Đoàn Văn Thắng — MSSV: 2A202602327

- **Vai trò thực hiện:** Đảm nhiệm toàn diện tất cả các khâu trong quy trình thực hành độc lập:
  - Cấu hình môi trường Docker CPU, nạp image container `pointpillars:kitti-cpu` và kiểm tra tính hợp lệ qua `smoke.json`.
  - Vận hành runner `student-bundle.py` thực hiện 3 lượt inference A, B, C và sinh các ca kiểm thử QC lỗi cao độ ($z$).
  - Trích xuất, kiểm tra đối chiếu dữ liệu giữa các tệp kết quả: `summary.csv`, các tệp tọa độ `boxes-*.json`, hình chiếu đứng `side-*.png`.
  - Phân tích cơ chế biến đổi tọa độ hình học, lập luận quyết định dừng batch / kiểm từng hộp và xây dựng báo cáo kỹ thuật.

- **Quan sát A/B/C có bằng chứng:**
  - *So sánh A và B ($\delta=0$ vs $\delta=1.73$ m, giữ nguyên pillar $0.16$ m):* Lượt A chỉ bắt được đúng 1 hộp duy nhất ở cự ly gần ($x \approx 8$ m, `mean_z = -0.124`), trong khi lượt B bắt được 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels với `mean_z = 0.382`). Đối chiếu trực quan trên ảnh `side-demo-delta-1.73-voxel-0.16.png`, các bounding box xe hơi ở dải $x \in [15, 45]$ m bao khít các cụm điểm phản xạ bám sát mặt đất, điều hoàn toàn không xuất hiện ở ảnh `side-demo-delta-0-voxel-0.16.png`. Điều này chứng minh việc bù trừ chiều cao cảm biến $\delta = 1.73$ m là thiết yếu để phân bố cao độ các điểm rơi đúng vào vùng kích hoạt của mạng trích xuất đặc trưng cột (Pillar Feature Net).
  - *So sánh B và C (pillar $0.16$ m vs $0.32$ m, giữ nguyên $\delta=1.73$ m):* Lượt C giảm số hộp từ 13 xuống còn 6 hộp (`mean_z = 0.514`). Đáng chú ý, toàn bộ 10 hộp xe hơi (`vehicles`) ở lượt B biến mất hoàn toàn, 6 hộp còn lại đều là `pedestrian`. Khi tăng kích thước cạnh pillar lên gấp đôi (diện tích ô tăng gấp 4 lần), độ phân giải không gian bị thô ráp, làm gộp nhiều điểm dị biệt vào cùng một cột và làm nhòe ranh giới hình học đặc trưng của xe hơi, khiến điểm tin cậy rơi xuống dưới ngưỡng score `0.3`.

- **Diễn giải phép z thuận và z ngược:**
  - *Biến đổi thuận (tiền xử lý dữ liệu trước inference):* $z_{model} = z_{source} - z_{ground} - \delta$. Thao tác này đưa cao độ đám mây điểm về hệ quy chiếu mặt đất chuẩn hóa theo cảm biến trước khi gom cụm vào các cột pillar. Việc thay đổi $\delta$ trước inference làm thay đổi trực tiếp cấu trúc phân bố điểm trong từng ô pillar, dẫn đến các vector đặc trưng học được hoàn toàn khác nhau. Đây là chạy lại mô hình trên dữ liệu đã biến đổi, tạo ra số hộp và vị trí phát hiện khác nhau.
  - *Biến đổi ngược (hậu xử lý tọa độ bounding box):* $z_{source} = z_{model} + \delta + z_{ground}$. Sau khi mô hình dự đoán tâm hộp $z_{model}$, ta bắt buộc phải cộng bù lại $\delta + z_{ground}$ để đưa bounding box trở lại hệ tọa độ thế giới thực của tệp PCD gốc. Nếu dịch hộp sau inference, đó đơn thuần là phép cộng tịnh tiến tọa độ đại số, không làm thay đổi số lượng hay đặc trưng hộp được phát hiện.

- **Quyết định lỗi batch và hành động xử lý:**
  - Đối với `case-batch-z` (13/13 hộp lệch $-(\delta + z_{ground}) \approx -3.46$ m): **DỪNG TOÀN BỘ BATCH (Stop Batch)** ngay lập tức. Toàn bộ 100% số hộp đều chìm sâu xuống dưới mặt đất cùng một độ lệch cố định. Đây là lỗi hệ thống (systemic pipeline error) do script xuất nhãn bỏ quên bước biến đổi ngược cao độ ($z_{source} = z_{model} + \delta + z_{ground}$). Tuyệt đối không import vào CVAT và không phân công chỉnh sửa thủ công từng hộp vì gây lãng phí công sức và hỏng phân bố dữ liệu huấn luyện. Cần dừng pipeline và chuyển giao cho kỹ sư tiền xử lý sửa code tọa độ.
  - Đối với `case-one-box-z` (1/13 hộp lệch z, 12 hộp còn lại chuẩn): **KIỂM TRA TỪNG HỘP (Inspect Per-Box)**. Pipeline chuyển đổi tọa độ vẫn hoạt động chính xác; hộp lỗi chỉ là cá biệt (outlier/false positive do phản xạ nhiễu hoặc điểm rơi ngoài phân bố). Người kiểm duyệt chỉ cần mở job trong CVAT để điều chỉnh lại độ cao hoặc xóa bỏ riêng hộp lỗi đó.
  - Đối với `case-correct`: Giữ nguyên hoặc kiểm tra đối chiếu thường quy, sẵn sàng cho công đoạn kiểm duyệt nhãn.

- **Điều chưa chắc chắn và giới hạn kỹ thuật:**
  - *Hạn chế của hình chiếu Side 2D ($x-z$):* Mặc dù ảnh Side trực quan hóa tốt cao độ $z$ và chiều dài $x$, nó hoàn toàn triệt tiêu trục $y$ và góc xoay hướng (yaw). Em chưa thể khẳng định chắc chắn góc yaw của xe hơi và người đi bộ nếu chưa đối chiếu qua góc nhìn Bird's Eye View (BEV) hoặc 3D viewer.
  - *Độ nhạy của $z_{ground}$ cục bộ:* Thuật toán ước lượng mặt đất hiện tại dựa trên giả định đường bằng phẳng. Trong các kịch bản thực tế khi xe đi vào dốc đứng, mố cầu hoặc đường gập ghềnh, $z_{ground}$ sai lệch sẽ kéo theo toàn bộ bounding box bị chênh vênh so với mặt đường thực tế.
  - *Kênh Reflectance/Intensity giả lập:* Bộ dữ liệu demo KITTI được gán intensity hằng (0.0 cho xe, 0.7 cho người/xe 2 bánh). Khi chuyển sang dữ liệu Robotaxi thực tế với cảm biến LiDAR đa dạng dải phản xạ, cần đánh giá lại độ nhạy của mô hình với ngưỡng score và phân bố intensity thật.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
