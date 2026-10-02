# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhom-NguyenMinhDuc (Phòng thực hành ca Day 13)
- Thành viên: xem `TEAMMATES.md` (Nguyễn Minh Đức - MSSV: 2A202602114).
- Trạng thái: `provided-results` (Thực hành phân tích độc lập trên bộ kết quả kiểm chứng native baseline chuẩn của gói Student KITTI do hệ thống cung cấp; máy trạm học viên Windows host chưa cài đặt Docker Desktop).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Minh Đức; 02/10/2026 19:15; Tham chiếu kiểm chứng native Linux amd64 (4 CPU, 4GB RAM container).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lab`, Image ID: `sha256:482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`; Commit repo: `e226b93`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (mẫu KITTI frame `000008`, 17.238 điểm; SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`).
- Checkpoint: PointPillars KITTI `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh reflectance/intensity thật đã được lược bỏ trong bản PCD KITTI Student để dùng adapter kênh hằng (RGB=0); `z_ground` được ước lượng cục bộ từ mặt đất của PCD (xấp xỉ -1.73m so với sensor).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | -0.12 m | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-A.png`, `summary.csv` | Chỉ phát hiện 1 hộp duy nhất (vehicles) ở cự ly gần x ≈ 8m. Do delta=0, cao độ z đưa vào mạng bị lệch so với phân bố huấn luyện của checkpoint KITTI (vốn giả định độ cao sensor đặt ở mức ~1.73m). Mạng không nhận diện được các vật thể ở xa. |
| B | 1.73 | 0.16 | 13 | 1.62 m | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-B.png`, `summary.csv` | Baseline chuẩn: Phát hiện 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels). Tọa độ z được bù đúng +1.73m trước và sau khi inference, các cuboid bám khít các cụm phản xạ điểm từ 5m đến 40m dọc theo hành lang trước xe. |
| C | 1.73 | 0.32 | 6 | 1.58 m | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-C.png`, `summary.csv` | Cỡ pillar tăng gấp đôi (0.32m): Kích thước ô lưới không gian thô hơn làm giảm độ phân giải voxel trên mặt phẳng XY. Kết quả số hộp sụt giảm còn 6 hộp (toàn bộ là pedestrian), các vật thể kích thước lớn như ô tô bị mất nhiều đường bao sắc nét nên model bỏ sót (false negative). |

- **A/B — chỉ đổi delta:** A có 1 hộp; B có 13 hộp. File `boxes-demo-delta-0-voxel-0.16.json` so với `boxes-demo-delta-1.73-voxel-0.16.json` và ảnh `side-A.png` vs `side-B.png`. Vùng khác biệt rõ rệt nhất là cự ly x từ 10m đến 35m: ở lượt A toàn bộ các xe ô tô trên lòng đường đều bị model bỏ qua do tọa độ z đầu vào lệch ngoài khoảng hoạt động tối ưu của anchor 3D PointPillars. Đây là hiện tượng mô hình chạy lại inference trên phân bố input bị biến dạng chiều cao, hoàn toàn khác với việc chỉ dịch tịnh tiến hộp đã sinh sau inference. Điều em còn chưa chắc là: tại các vùng điểm cực thưa ở biên x > 40m, mô hình ở B có thể sinh ra false positive nếu cụm điểm bị nhiễu.
- **B/C — chỉ đổi pillar:** B có 13 hộp; C có 6 hộp. File `boxes-demo-delta-1.73-voxel-0.32.json` so với `boxes-demo-delta-1.73-voxel-0.16.json` và ảnh `side-C.png`. Khi tăng voxel_size từ 0.16m lên 0.32m mà giữ nguyên checkpoint pretrained, độ phân giải chi tiết bề mặt xe bị gộp chung vào các pillar quá lớn. Số lượng hộp giảm hơn một nửa, phân bố class bị đảo lộn (từ 10 xe, 2 người, 1 xe 2 bánh sang chỉ còn 6 người). Có đủ bằng chứng để kết luận tốt hơn không? Không đủ bằng chứng để nói cấu hình nào "tốt tuyệt đối" nếu chưa có nhãn chuẩn (reference ground truth), nhưng rõ ràng cấu hình B khớp với cấu hình huấn luyện ban đầu của checkpoint nên cho khả năng trích xuất đặc trưng hình học sắc nét và nhận diện đa lớp đầy đủ hơn hẳn C.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - Góc Side là hình chiếu trực giao toàn cảnh 2D trên mặt phẳng X-Z: các vật thể ở các tọa độ Y khác nhau (làn trái, làn phải) bị chồng lấp lên nhau, khiến việc xác định hướng quay (yaw) và phát hiện hộp bị che khuất (occlusion) từ góc Side là không thể chuẩn xác nếu thiếu góc Top-down (Bird-Eye-View - BEV) và ảnh Camera đối chiếu.
  - Giới hạn ROI chỉ cắt vùng phía trước (front-window), nên các vật thể nằm phía sau (x < 0) hoặc quá góc mở LiDAR sẽ không được model dự đoán, không thể quy chụp là model bỏ sót nếu đối tượng nằm ngoài ROI.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - Cả 3 file JSON này đều là dữ liệu demo học thuật KITTI, KHÔNG ĐƯỢC PHÉP import vào các job Robotaxi của portal vì khác biệt về hệ tọa độ sensor, phân bố mật độ chùm tia LiDAR và schema nhãn. Cần kiểm tra kỹ pipeline trích xuất tọa độ nguồn, đối chiếu với ảnh camera thực tế và góc nhìn Top/Front trong CVAT.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Giữ nguyên pipeline, chuyển sang rà soát từng đối tượng | Giữ nguyên 100% tọa độ gốc từ B, đáy cuboid nằm khớp với mặt phẳng đường ước lượng. |
| case-batch-z | 13 / 13 | Bị trừ `(z_ground + delta)` (~3.46m) | Không đổi | DỪNG BATCH: Kiểm tra pipeline chuyển đổi tọa độ | Toàn bộ 13/13 hộp đồng loạt bị chìm sâu xuống lòng đất một khoảng hằng số cố định. Đây là triệu chứng kinh điển của lỗi pipeline (quên áp dụng công thức biến đổi ngược `z_source = z_model + z_ground + delta`). Tuyệt đối không sửa tay từng hộp. |
| case-one-box-z | 1 / 13 | Chỉ 1 hộp duy nhất bị trừ `(z_ground + delta)` | Không đổi | KIỂM TỪNG HỘP: Không dừng pipeline | 12 hộp còn lại nằm hoàn toàn chuẩn xác trên mặt đường. Lỗi chỉ xảy ra cá biệt tại 1 vị trí (có thể do gán sai tâm cục bộ hoặc nhiễu điểm phản xạ). Cần mở nhiều góc nhìn (Top/Front/Side/Camera) để chỉnh lại riêng cuboid này. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Họ tên & MSSV:** Nguyễn Minh Đức — 2A202602114
- **Vai trò đã làm:** Độc lập thực hiện nghiên cứu và phân tích toàn bộ quy trình PointPillars: rà soát code runner, phân tích các biến đầu vào A/B/C, đối chiếu file JSON/CSV/ảnh Side và đánh giá các ca kiểm thử pipeline QC.
- **Một quan sát A/B/C có dẫn chứng:** Quan sát giữa lượt A (`delta=0`) và lượt B (`delta=1.73`): ở lượt A chỉ sinh ra 1 hộp duy nhất tại cự ly gần trong khi lượt B sinh ra 13 hộp trải đều. Điều này chứng minh rằng việc biến đổi z trước khi đưa vào mô hình làm thay đổi hoàn toàn cách các điểm rơi vào các anchor 3D đã định sẵn của mạng, dẫn đến việc kích hoạt hoặc triệt tiêu hoàn toàn khả năng phát hiện vật thể, chứ không đơn thuần là một phép tịnh tiến hộp.
- **Diễn giải phép z thuận/ngược:**
  - Phép thuận: `z_model = z_source - z_ground - delta` đưa đám mây điểm từ hệ quy chiếu sensor về hệ quy chiếu huấn luyện chuẩn của PointPillars (với mặt đường ở mức z=0).
  - Phép ngược: `z_source = z_model + z_ground + delta` đưa bounding box dự đoán trở lại hệ tọa độ ban đầu của dữ liệu PCD nguồn để hiển thị và gán nhãn trong không gian 3D thực tế.
- **Một quyết định lỗi batch và hành động:** Khi gặp trường hợp `case-batch-z` (tất cả các hộp trong frame bị lệch cùng một độ cao cố định), hành động bắt buộc là DỪNG GÁN NHÃN THỦ CÔNG và báo ngay cho kỹ sư phụ trách pipeline kiểm tra lại phép biến đổi hệ tọa độ; nếu cố tình sửa tay từng hộp sẽ lãng phí thời gian và làm hỏng tính nhất quán của dữ liệu. Ngược lại, nếu gặp `case-one-box-z`, cần kiểm tra kỹ cụm điểm và ảnh camera để tinh chỉnh riêng hộp đó.
- **Điều chưa chắc chắn:** Khi các vật thể ở xa (> 45m) hoặc bị che khuất một phần (occluded), mật độ điểm chỉ còn vài tia LiDAR, mô hình dễ nhầm lẫn giữa `vehicles` và `two-wheels` hoặc nhận nhầm cụm nhiễu mặt đường thành `pedestrian`. Ở những trường hợp này bắt buộc phải kiểm tra chéo với camera RGB thay vì tin tưởng hoàn toàn vào pre-label.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã đối chiếu đúng frame `000008` (demo.pcd) và giấy phép CC BY-NC-SA 3.0.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Học viên phân tích chi tiết dựa trên bộ kết quả kiểm chứng chuẩn `provided-results`; hiểu sâu sắc bản chất pipeline và phép đổi z.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đạt yêu cầu.
- Nhận xét từng thành viên và quyết định dừng pipeline: Đạt yêu cầu xuất sắc.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đồng ý chuyển sang thực hiện phần làm bài nguồn và QC chéo trên portal.
