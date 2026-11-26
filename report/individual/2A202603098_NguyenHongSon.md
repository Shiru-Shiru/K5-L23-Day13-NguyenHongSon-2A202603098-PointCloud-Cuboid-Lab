# Báo cáo Day 13 — Nguyễn Hồng Sơn, 2A202603098

## 1. Kết luận trước
- **Thông tin chung:** MSSV `2A202603098`, task `D13-2A202603098`, gồm 10 frame: `s01f07`, `s01f48`, `s02f17`, `s02f35`, `s03f20`, `s03f59`, `s04f39`, `s04f59`, `s05f13`, `s05f38`. Môi trường làm bài: hệ điều hành Windows, trình duyệt Chrome/Edge, CVAT bản lab 2.82/2.83; MSSV bạn ngồi cạnh: `[điền MSSV bạn ngồi cạnh]`.
- **Tổng quan kết quả:** Tổng cộng đã gán **69 cuboid** trên 10 frame (`s01f07`: 11, `s01f48`: 10, `s02f17`: 8, `s02f35`: 5, `s03f20`: 6, `s03f59`: 5, `s04f39`: 6, `s04f59`: 5, `s05f13`: 7, `s05f38`: 6); các frame ít nhất là `s02f35`, `s03f59`, `s04f59` (5 cuboid/frame) do cảnh thoáng hơn hoặc có xe lớn/vật ở xa bị che khuất. Phiếu tự kiểm `selfcheck` gắn cờ **15/69 cuboid** (chủ yếu là cảnh báo kích thước hơi vượt ngưỡng Car/Bus/Cyclist và 2 `Cyclist` ở `s01f48` có `ground_gap_m` cao). Phần tự tin nhất là xác định class và hướng `yaw_deg` (đã đối chiếu qua ảnh camera); phần còn yếu nhất là căn sát kích thước chiều rộng/dài với các xe ở xa (> 25 m) và căn đáy `z_bottom` ở những vùng ít điểm mặt đường.
- **Cách dùng CVAT:**
  - *Đổi góc nhìn:* kéo chuột trái để xoay khung 3D, kéo chuột phải để dời khung nhìn, lăn chuột để phóng to/thu nhỏ, nháy đúp vào vùng trống để về góc nhìn mặc định; xoay thẳng từ trên xuống để quan sát toàn cảnh 360°.
  - *Vẽ và chỉnh cuboid:* chọn công cụ Cuboid → chọn class → bấm **Shape** (không dùng Track), đưa tới cụm điểm rồi nháy đúp để đặt; chọn cuboid để chỉnh tâm/dài/rộng/yaw trên khung **Top** và chỉnh cao độ/đáy chạm đường trên khung **Side** và **Front** (không xoay ở Side/Front).
  - *Hướng và kiểm tra trên ảnh:* bật **Appearance → Cuboid orientation** để thấy vạch đỏ `+X` chỉ về đầu xe, rồi chạy `python scripts/lab.py preview` để chiếu cuboid lên ảnh camera trong `work/xem/` kiểm tra lại.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ (`s01f07`, `s01f48`). Ảnh: `t1-s01f07-3d.png`, `t1-s01f07-top.png`, `t1-s01f48-3d.png`, `t1-s01f48-top.png`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s01f07` | Phía trước (`−y`) ở **cạnh trên** ảnh nhìn từ trên xuống; `+x` chỉ sang **trái**, `+y` chỉ xuống **dưới** (phía sau xe) | Phẳng, thấp hơn LiDAR khoảng `1,7 m` (`z ≈ −1,7 m`) | Khoảng `38–40 m` (cuboid `idx 11` ở `37,97 m`) | `Car`, `Truck`, hàng cây, dải phân cách, cột đèn/biển báo | Phía sau (`+y`) có các xe con (`idx 2, 4, 10, 11`); bên trái (`+x`) có làn xe ngược chiều (`idx 5–9`) và xe con xanh trên `cam05` (`idx 1`) | Một số cụm điểm thưa trên vỉa hè/sau tán cây và các xe ngoài `40 m` chỉ còn ít điểm rời rạc |
| `s01f48` | Phía trước (`−y`) ở **cạnh trên** ảnh nhìn từ trên xuống; `+x` chỉ sang **trái**, `+y` chỉ xuống **dưới** (phía sau xe) | Phẳng, thấp hơn LiDAR khoảng `1,7 m` (`z ≈ −1,7 m`) | Khoảng `26–30 m` (cuboid `idx 3` ở `26,04 m`) | `Car`, `Cyclist`, giá long môn biển báo, cây cối | Phía sau (`+y`) có 2 `Car` (`idx 6, 10`); bên phải (`−x`) có 5 `Cyclist` (`idx 1–5`) di chuyển dọc lề phải | Cụm điểm thưa trên vỉa hè bên phải sau hàng cây và các xe phía trước bị hai xe con ngay trước mặt che khuất |

- **3 điều chỉ camera cho biết** (từ `s01f07` và `s01f48`):
  1. Màu sắc, đặc điểm nhận dạng đầu/đuôi xe (đèn pha trắng phía trước, đèn hậu đỏ phía sau) để xác định hướng `yaw`.
  2. Phân biệt rõ các class dễ nhầm trên point cloud (ví dụ xe máy/xe đạp có người lái hay không, xe bán tải hay SUV, biển báo/giá long môn).
  3. Tình trạng vật bị che khuất một phần phía sau xe khác (ví dụ đầu xe buýt hoặc xe ở làn bên cạnh qua `cam05`).
- **3 điều chỉ LiDAR cho biết** (từ `s01f07` và `s01f48`):
  1. Tọa độ và khoảng cách thực theo mét (`x, y, z`, `dist_m`) tới mọi vật xung quanh `360°`, kể cả góc mù giữa các camera.
  2. Chiều cao, bề rộng, chiều dài thực tế và cao độ mặt đường (`z ≈ −1,7 m`) để dựng cuboid đúng kích thước 3D.
  3. Cấu trúc hình học của vật (mặt hông, nóc xe, bóng che phía sau vật cản) giúp tách các vật đứng gần nhau.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| `s02f17` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Có người đi bộ (`Pedestrian`) ở khoảng `22,37 m` với số điểm ít (`55` điểm) |
| `s02f35` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Có `Cyclist` ở `21,50 m` bên phải phía sau |
| `s03f20` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Giao lộ có các xe quay ngang/chuyển hướng (`yaw_deg` khoảng `9°–13°` và `80°–82°`) |
| `s03f59` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Có 2 xe buýt lớn (`Bus`), trong đó 1 xe ở rất xa (`52,61 m`, vượt ngưỡng `50 m`) |
| `s04f39` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Nhiều xe con chạy hai chiều dọc theo trục `y`, mật độ điểm rõ ở trong `30 m` |
| `s04f59` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Cảnh khá thoáng, chủ yếu xe con (`Car`) ở hai bên và phía trước/sau |
| `s05f13` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Có xe con bị che khuất một phần (`occluded = 1`, `idx 6` ở `11,19 m`) |
| `s05f38` | Cạnh trên (`−y` hướng lên, `+x` sang trái) | Có 1 `Truck` ở `29,62 m` và 1 xe dài `8,63 m` (`idx 5`) ở `38,97 m` bị che một phần |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s01f07, idx 1` | `Car` | Đúng (`270,5°`, cùng chiều xe thu dữ liệu) | Chạm đường (`z_bottom = −1,993 m`, lệch `−0,21 m`) | Đã kiểm Top/Side/Front và ảnh `cam05` (`t2-a-car.png`) | `l = 5,128 m, w = 2,121 m, h = 1,767 m` (gần chuẩn SUV/van, rộng hơn `2,10 m` khoảng `0,02 m`) |
| (b) xe buýt hoặc xe tải | `s03f59, idx 4` | `Bus` | Đúng (`270,9°`, theo hướng di chuyển) | Chạm đường (`z_bottom = −1,687 m`, lệch `+0,06 m`) | Đã kiểm Top/Side/Front và ảnh `cam05` (`t2-b-bus-truck.png`) | `l = 12,832 m, w = 3,067 m, h = 3,470 m` (dài và rộng hơi lớn hơn chuẩn xe buýt thường `10,0–12,5 m × 2,50–2,60 m`) |
| (c) người đi bộ hoặc xe hai bánh | `s01f48, idx 1` | `Cyclist` | Đúng (`267,3°`, hướng thẳng về trước) | Hơi cao (`z_bottom = −1,331 m`, lệch `+0,16 m`) | Đã kiểm Top/Side/Front và ảnh (`t2-c-ped-cyclist.png`) | `l = 1,816 m, w = 0,695 m, h = 1,740 m` (nằm trong khoảng xe đạp/xe máy có người bảng 4.3) |

- **Vòng hiệu chuẩn & vật bỏ qua:** Trong vòng hiệu chuẩn, các cuboid ban đầu dễ lệch nhau ở việc ước lượng kích thước chiều dài phần bị che và độ cao đáy `z_bottom`; cả lớp chốt thống nhất theo guideline: vạch đỏ `+X` luôn hướng về đầu xe/hướng di chuyển, đáy hộp căn chạm mặt đường ở khung Side/Front, và kích thước đối chiếu bảng 4.3 (không bao dư điểm nhiễu). Các vật bỏ qua vì không đủ cơ sở: các cụm điểm quá thưa ở khoảng cách xa `> 40–50 m` sau tán cây hoặc lề đường ở `s01f07` và `s01f48` (nghi vấn `LOW_VISIBILITY` / `UNCERTAIN_SCOPE`, không đạt `15–20%` thể tích rõ ràng nên không vẽ).

## 4. Bốn hiện tượng chất lượng (Bước 4)
Nguyên nhân ghi là **giả thuyết**.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `s01f07` / `t4-1-bong-che.png` / task cá nhân `D13-2A202603098` | **Giả thuyết:** các xe gần hoặc xe lớn chắn tia laser từ LiDAR khiến phía sau chúng tạo thành vùng khuất | Phía sau vật chắn chỉ thấy một phần góc/mặt bên của xe bị che (`idx 8 Truck`, `idx 9 Car`), phía sau nữa mất hẳn điểm mặt đường | Bật thuộc tính **occluded**, cuboid vẫn bao trọn phần kích thước suy luận được theo bảng 4.3 và ảnh camera |
| 2. Thưa ở xa | `s03f59` / `t4-2-thua-o-xa.png` / task cá nhân `D13-2A202603098` | **Giả thuyết:** mật độ tia LiDAR phân kỳ theo khoảng cách (`dist_m = 52,61 m` với `Bus idx 5`), nên số tia trúng vật giảm mạnh | Xe buýt chỉ còn `208` điểm rời rạc theo các đường quét thưa, khó thấy rõ mép đầu/đuôi và mặt đáy | Chỉ vẽ khi thấy được khoảng `15–20%` thể tích và đủ cơ sở từ ảnh camera; nếu không đủ cơ sở thì bỏ qua và ghi `LOW_VISIBILITY` (với vật `> 50 m` không tính điểm) |
| 3. Phản xạ thấp / ít điểm bất thường | `s01f07` / `t4-3-phan-xa-thap.png` / task cá nhân `D13-2A202603098` | **Giả thuyết:** bề mặt xe tối màu, kính chắn gió hoặc góc tới tia laser quá xiên làm cường độ phản xạ yếu, nhiều tia không hồi về | Trên thân xe hoặc phần kính/nóc xe xuất hiện các khoảng trống ít điểm dù vật không ở quá xa | Ghi nhận là giả thuyết, không thu hẹp cuboid theo phần điểm thiếu; dùng phần khung còn lại và bảng 4.3 để bao đủ thể tích xe |
| 4. Điểm lạ | `s01f07` / `t4-4-diem-la.png` / task cá nhân `D13-2A202603098` | **Giả thuyết:** nhiễu đa đường phản xạ, bụi/lá cây hoặc điểm rời rạc từ kết cấu lân cận xung quanh xe | Xuất hiện các điểm lẻ tẻ lơ lửng hoặc tách rời khỏi hình dạng chữ L/chữ nhật đặc trưng của thân xe | Không gán các cụm điểm lạ thành vật; khi chỉnh 6 mặt hộp phải loại trừ điểm nhiễu, không kéo giãn cuboid bao theo |

## 5. Số từ cuboid của tôi (Bước 5)
Lấy từ `submission/D13-2A202603098-bangA.csv` và `submission/D13-2A202603098-tukiem.csv`.

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| `s01f07, 1, Car` | `−1,110 m` | `1,767 m` | `−1,993 m` | `3,77 m` | `3,93 m` | `270,5°` | Chạm đường (`ground_gap_m = −0,21 m`, lệch trong ngưỡng `0,3 m`) |
| `s01f07, 9, Car` | `−1,287 m` | `2,757 m` | `−2,666 m` | `31,95 m` | `31,97 m` | `92,3°` | Chìm (`ground_gap_m = −0,30 m`, đáy thấp hơn mặt đường cục bộ) |
| `s01f48, 4, Cyclist` | `−0,481 m` | `1,434 m` | `−1,198 m` | `18,81 m` | `18,82 m` | `267,7°` | Lơ lửng (`ground_gap_m = +0,58 m`, cao hơn mặt đường `> 0,3 m`) |

- **(a) Cuboid gần nhất và xa nhất:**
  - Gần nhất: `s01f07, idx 1 (Car)` có `dist_m = 3,77 m`, `r = 3,93 m`.
  - Xa nhất: `s03f59, idx 5 (Bus)` có `dist_m = 52,61 m`, `r = 52,61 m` (lưu ý cuboid này vượt ngưỡng `50 m` nên không được tính khi chấm tự động).
- **(b) So sánh kích thước với bảng 4.3:**
  - `Pedestrian` (`s02f17, idx 8`): `l = 0,528 m, w = 0,776 m, h = 1,784 m` — cả 3 chiều đều nằm trong khoảng chuẩn *Người lớn* (`0,4–0,9 m × 0,5–0,9 m × 1,50–1,95 m`). (Ngoài ra, `Cyclist` `s02f35, idx 5` có `l = 1,812 m, w = 1,272 m, h = 1,630 m`, chiều rộng `1,272 m` vượt ngưỡng `0,3–1,2 m` do góc nhìn thưa ở `21,50 m` khiến hộp bị kéo rộng).
  - `Car` (`s01f07, idx 2`): `l = 6,157 m, w = 2,009 m, h = 1,969 m` — chiều dài `6,157 m` nằm ngoài khoảng `3,8–5,3 m` (và ngưỡng tự kiểm `3,0–6,0 m`) do xe ở phía sau (`19,79 m`) hơi thưa điểm ở phần đuôi/đầu nên khi kéo hộp đã bao dư chiều dài.
- **(c) Hướng `yaw_deg`:**
  - Xe cùng chiều phía trước: `s01f07, idx 3 (Car)` ở `y = −19,205 m` có `yaw_deg = 270,6°` (rất gần `270°`).
  - Xe đi ngược chiều: `s01f07, idx 6 (Car)` ở làn đối diện (`x = 18,310 m`) có `yaw_deg = 91,0°` (rất gần `90°`).

Mật độ theo khoảng cách (4 cuboid `Car` ở các `dist_m` khác nhau):

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| `s01f07, 1, Car` | `3,77 m` | `3221` |
| `s01f07, 6, Car` | `18,43 m` | `668` |
| `s01f07, 7, Car` | `24,80 m` | `404` |
| `s01f07, 11, Car` | `37,97 m` | `90` |

- **Quy luật mật độ:** Khi khoảng cách tăng gấp đôi từ `18,43 m` ra `37,97 m`, số điểm trên `Car` giảm từ `668` xuống `90` (chỉ còn khoảng `1/7`). So với đường cong trên slide bài giảng (xe con trung bình: `1968` điểm ở `0–10 m`, `718` ở `10–20 m`, `214` ở `20–30 m`, `72` ở `30–40 m`, `19` ở `60–80 m`), xu hướng giảm nhanh theo khoảng cách hoàn toàn tương đồng (nhóm `10–20 m` đạt `668` điểm gần mức `718`, nhóm `30–40 m` đạt `90` điểm gần mức `72`).
- **Phần tự kiểm:**
  - *Đã sửa:* Sau khi chạy `preview` và `selfcheck`, đã kiểm tra lại hướng đầu/đuôi và chỉnh lại một số hộp `Car` ở `s01f07` và `s01f48` cho khớp thân xe trên ảnh camera, đồng thời xác nhận `z_bottom` của các xe gần chạm mặt đường.
  - *Cảnh báo giữ nguyên:* Giữ nguyên các cảnh báo kích thước hơi dư biên trên các xe bị che hoặc ở xa (`s01f07 idx 5, 6, 7, 9`, `s02f17 idx 4, 5`, `s03f59 idx 5 Bus`) vì point cloud thực tế bị che khuất/thưa điểm và vẫn cần giữ kích thước bao đủ hình dáng suy luận; giữ nguyên `s01f48 idx 4, 5 Cyclist` để ghi nhận trung thực sai số cao độ đáy ở vùng ít điểm mặt đường.
- **Đổi kiểm:** MSSV người kiểm chéo: `[điền MSSV bạn ngồi cạnh]`. Hai con số đối chiếu với `submission/D13-2A202603098-bangA.csv`: `s01f07, idx 1` (`dist_m = 3,77 m`) và `s03f59, idx 4` (`length = 12,832 m`), cả hai đều khớp với bảng A đã xuất từ CVAT.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- **(a) Từ phía model học trên nhãn 3D:**
  1. Model nhận thức cho xe tự lái cần nhãn 3D đo bằng **mét** (tọa độ `x, y, z` và kích thước `l, w, h` theo hệ tọa độ thực quanh xe) thay vì pixel 2D vì bộ lập kế hoạch chuyển động cần biết chính xác khoảng cách an toàn, vận tốc và không gian chiếm chỗ trong thế giới thực (`3D`), điều mà ảnh 2D bị biến dạng phối cảnh không cung cấp trực tiếp được.
  2. Vật quá thưa điểm cần quy tắc riêng (`15–20%` thể tích / `LOW_VISIBILITY`) để tránh việc người gán nhãn "đoán mò" tạo ra nhãn giả (nhãn nhiễu tâm, sai kích thước hoặc sai class), làm model học sai phân phối đặc trưng LiDAR ở tầm xa.
- **(b) Cách dùng AI:**
  - *Công cụ:* Antigravity.
  - *Câu hỏi chính:* Hỏi quy trình thực hiện lab, lệnh PowerShell, quy ước hướng cuboid.
  - *Cách dùng kết quả:* Dùng để đối chiếu công thức tính (`z_bottom`, `r`), kiểm tra định dạng bảng và rà soát lỗi trước khi chạy `check_submission.py`; toàn bộ cuboid trên CVAT và ảnh chụp màn hình đều tự thực hiện và kiểm tra trực tiếp.
- **(c) Điều còn chưa chắc:** Có những trường hợp xe ở quá xa (như xe ở `> 30–50 m`) rất ít điểm LiDAR nên khó xác định chính xác tuyệt đối mép đầu/đuôi và độ cao đáy chạm đường; ngoài ra một số cuboid `Car` đang có `width`/`length` hơi lớn hơn ngưỡng bảng 4.3 nếu có thêm thời gian sẽ tinh chỉnh sát hơn ở khung Top.
