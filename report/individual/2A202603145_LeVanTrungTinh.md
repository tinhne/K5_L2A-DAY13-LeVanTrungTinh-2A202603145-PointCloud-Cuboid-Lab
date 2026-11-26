# Báo cáo Day 13 — Lê Văn Trung Tính, 2A202603145

## 1. Kết luận trước
- Mã số: 2A202603145 · task `D13-2A202603145` · 10 frame: s01f16, s01f44, s02f17, s02f36, s03f21, s03f59, s04f07, s04f34, s05f03, s05f44
- macOS 27 · trình duyệt: Google Chrome · CVAT server 2.74.1 (máy cá nhân, cổng 8081; cvat-sdk 2.83.0) · MSSV bạn ngồi cạnh: 2A202600118

Tổng số cuboid đã gán nhãn: 103 cuboid (đạt yêu cầu tối thiểu ≥ 53 cuboid, vượt mức khuyến nghị 60–70 hộp). Phân bố theo từng frame: s01f16 (7), s01f44 (11), s02f17 (12), s02f36 (6), s03f21 (14), s03f59 (10), s04f07 (14), s04f34 (9), s05f03 (11), s05f44 (9). Frame có ít cuboid nhất là s02f36 với 6 cuboid (lý do: mật độ giao thông trên đoạn đường này khá thưa thoáng, ít xe lưu thông trong phạm vi 50m quanh xe thu dữ liệu). Sau khi chạy `python scripts/lab.py selfcheck`, có 9/103 cuboid bị cờ cảnh báo (chủ yếu do xe bị che khuất một phần nên kích thước quan sát được ngắn hơn bảng chuẩn, hoặc xe đỗ sát lề đường/vỉa hè nâng cao).
- Phần tự tin nhất: Nhận diện và gán nhãn xe con (Car) ở cự ly gần và trung bình (< 25m), căn sát các mặt tiếp giáp rõ của cụm điểm trên cả 3 khung nhìn Top/Side/Front, xác định đúng hướng đầu xe (yaw_deg) và độ cao chạm mặt đường.
- Phần còn yếu nhất: Ước lượng kích thước chuẩn của các vật thể ở xa (> 35m) do mật độ điểm LiDAR bị suy giảm mạnh, và các trường hợp xe bị che khuất một phần (occluded) cần dựa nhiều vào suy đoán từ ảnh camera.

Trong khung 3D, chuyển sang nhìn từ trên xuống để định vị vật; xoay và dời góc nhìn để kiểm tra các mặt. Tạo cuboid dạng Shape, nháy đúp để chỉnh rồi đối chiếu lần lượt Top, Side, Front. Bật Cuboid orientation để xem hướng; dùng `python scripts/lab.py preview` để đối chiếu cuboid với ảnh camera.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ: s01f16 và s01f44. Ảnh: `t1-s01f16-3d.png`, `t1-s01f16-top.png`, `t1-s01f44-3d.png`, `t1-s01f44-top.png` (ảnh nhìn từ trên xuống, có đánh dấu phía trước, +x, +y).

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| s01f16 | Phía trước xe hướng về cạnh dưới bên phải (hướng −y); +x chỉ sang bên trái xe (cạnh trên bên phải), +y chỉ về phía sau đuôi xe (cạnh trên bên trái) | Mặt đường bằng phẳng; thấp hơn LiDAR khoảng 1,7 m (z đáy các xe quanh −1,70 m) | Xe buýt lớn phía trước bên phải (~30,9 m) và xe con phía xa (~31,4 m) | Car, Bus | Phía sau có 2 xe con bám đuôi (cách 17,8 m và 20,1 m); bên phải có xe con cùng chiều | Cụm dải phân cách cây xanh và tán cây rậm rạp bên lề đường |
| s01f44 | Phía trước xe hướng về cạnh phải (hướng −y); +x chỉ sang bên trái xe (cạnh trên), +y chỉ về phía sau đuôi xe (cạnh trái) | Mặt đường bằng phẳng; thấp hơn LiDAR khoảng 1,75 m (z đáy trung bình quanh −1,65 m đến −1,84 m) | Xe buýt lớn phía trước (~38,8 m) và xe con đi trước (~34,4 m) | Car, Bus | Phía sau có 2 xe con chạy sau (cách 13,1 m và 26,4 m); hai bên có các xe con song hành | Cụm phản xạ kính từ tường nhà cao tầng và cột biển báo ở rìa xa |

Camera cho biết màu sắc, chữ/đặc điểm nhận dạng trực quan và ngữ cảnh che khuất; LiDAR cho biết tọa độ 3D theo mét, khoảng cách/hình dạng không gian và độ cao tương đối với mặt đường. Ba ví dụ cụ thể đối chiếu giữa hai nguồn dữ liệu:
1. Ở frame s01f16: Camera cam01 cho thấy xe buýt phía trước màu trắng-vàng và xe con màu trắng, biển số và đèn hậu rõ ràng; trong khi LiDAR cho biết chính xác xe buýt cách xe mình 30,85 m, dài 12,56 m, cao 3,64 m và sàn xe cách mặt đất 0,10 m.
2. Ở frame s01f44: Camera cam03/cam05 cho biết xe con bên cạnh màu xanh đậm đang chạy cùng chiều trên làn đường nhựa; trong khi LiDAR đo đạc chính xác tâm xe cách 6,56 m, hướng xoay yaw 271,2° và đáy xe chạm đường ở z = −1,83 m.
3. Ở frame s01f16 và s01f44: Phía sau xe thu dữ liệu hoàn toàn không xuất hiện trên camera trước (cam01), nhưng camera sau (cam09) cho thấy hình ảnh xe bám đuôi và LiDAR ghi nhận đầy đủ 2 cụm điểm xe bám sau cách lần lượt 17,78 m và 20,10 m với bề rộng thân xe 2,09 m.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| s02f17 | Hướng −y (chỉ về cạnh dưới của màn hình) | Có người đi xe hai bánh (Cyclist) di chuyển sát sườn xe bên trái (cách 7,06 m); làn đối diện có xe chạy ngược chiều |
| s02f36 | Hướng −y (chỉ về cạnh dưới của màn hình) | Mật độ xe thưa thoáng; các xe làn ngược chiều có góc yaw quanh 90° |
| s03f21 | Hướng −y (chỉ về cạnh dưới của màn hình) | Khu vực giao lộ ngã tư rộng có vạch người đi bộ; bên trái có hàng xe đỗ chéo góc (yaw ~ 15°–18°) |
| s03f59 | Hướng −y (chỉ về cạnh dưới của màn hình) | Xe buýt lớn đi ngay phía trước (cách ~20,4 m) che khuất một phần lớn góc nhìn của camera trước cam01 |
| s04f07 | Hướng −y (chỉ về cạnh dưới của màn hình) | Xe tải lớn (Truck) đi bên phải phía trước (cách 14,18 m) và dải rào chắn vàng kéo dài dọc đường |
| s04f34 | Hướng −y (chỉ về cạnh dưới của màn hình) | Người đi xe hai bánh (Cyclist) đi sát bên trái xe (cách 8,75 m); đoàn xe phía trước kéo dài đến gần 50 m |
| s05f03 | Hướng −y (chỉ về cạnh dưới của màn hình) | Xe con chạy rất sát bên sườn trái (cách 3,39 m, gần nhất bài lab); phía xa có người đi xe hai bánh (cách 46,59 m) |
| s05f44 | Hướng −y (chỉ về cạnh dưới của màn hình) | Đoạn đường cong chuyển hướng; xe buýt ở xa (cách 37,15 m) và cụm điểm xe đạp ở xa rất thưa |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | s01f44, idx 1 | Car | Đúng (yaw = 268,5°, mũi tên đỏ hướng về đầu xe cùng chiều xe chạy) | Đúng (z đáy = −1,787 m, sát mặt đường phẳng) | Đã kiểm đủ 3 mặt chiếu và đối chiếu cam01, cam03 | Đúng (L = 4,32 m, W = 1,98 m, H = 1,55 m, nằm trong khoảng chuẩn) |
| (b) xe buýt hoặc xe tải | s04f07, idx 13 | Truck | Đúng (yaw = 82,1°, mũi tên hướng về cabin đầu xe tải) | Đúng (z đáy = −1,649 m, chạm mặt đường) | Đã kiểm đủ 3 mặt chiếu (thấy rõ cabin và thùng hàng) và cam05 | Đúng (L = 6,96 m, W = 2,61 m, H = 2,99 m, chuẩn xe tải) |
| (c) người đi bộ hoặc xe hai bánh | s04f34, idx 9 | Cyclist | Đúng (yaw = 269,7°, mũi tên hướng về phía trước theo hướng lái) | Đúng (z đáy = −1,461 m, chạm mặt đường gờ sát lề) | Đã kiểm đủ 3 mặt chiếu và đối chiếu cam06, cam07 | Đúng (L = 2,28 m, W = 0,76 m, H = 1,37 m, chuẩn xe hai bánh) |

Vòng hiệu chuẩn:
- Khi so sánh với kết quả các bạn trong lớp ở task chung `D13-HC485087`, độ lệch ban đầu chủ yếu xuất hiện ở việc ước lượng chiều dài xe (length) khi vật bị che khuất một phần (một số bạn kéo hộp quá ngắn chỉ bao quanh phần điểm nhìn thấy), và độ cao đáy hộp (nhiều bạn để đáy hộp bị chìm sâu xuống dưới mặt đường hơn 0,3 m).
- Cả lớp đã thống nhất cách hiểu theo guideline:
  1. Vật bị che một phần (`occluded`): cuboid phải bao trùm kích thước thực tế suy đoán được của vật theo tỷ lệ tối thiểu 15–20% thể tích và bật cờ `occluded`.
  2. Mũi tên yaw (trục +X của cuboid) luôn phải hướng về phía trước đầu xe, căn cứ kết hợp ảnh camera tương ứng để xác định hướng đầu xe.
  3. Đáy cuboid phải chạm mặt đường (z_bottom xấp xỉ −1,7 m ± 0,2 m), kiểm tra đối chiếu kỹ trên cả hai khung nhìn Side và Front.
- Các vật bỏ qua vì không đủ cơ sở:
  - Frame s03f21, tọa độ x ≈ 35 m, y ≈ −45 m: Cụm điểm rải rác dưới 5 điểm, ảnh camera bị bóng râm che khuất, không xác định được hình dạng hình học -> Ghi nhận `LOW_VISIBILITY` và bỏ qua không gán nhãn.
  - Frame s05f44, vị trí lề đường bên phải ở khoảng cách > 45 m: Cụm điểm thưa thớt không phân biệt được là người đi bộ hay cột mốc/bụi cây -> Ghi nhận `UNCERTAIN_CLASS` và không gán cuboid.

## 4. Bốn hiện tượng chất lượng (Bước 4)
Nguyên nhân ghi là **giả thuyết**. Frame mượn thì ghi rõ nguồn: task chung, hoặc MSSV của bạn cho mượn.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | Frame s01f44 · `t4-1-bong-che.png` | Tia laser LiDAR bị thân xe buýt và xe con phía trước chắn hoàn toàn, không thể xuyên qua để quét các vật thể phía sau | Khoảng trống không có điểm nào hình rẻ quạt kéo dài ngay sau đuôi xe buýt và xe con | Không được coi vùng trống là đường trống; vật bị che khuất một phần nếu đủ cơ sở thì vẽ bao trùm và bật cờ `occluded`, nếu che hoàn toàn thì không vẽ |
| 2. Thưa ở xa | Frame s01f44 · `t4-2-thua-o-xa.png` | Do góc mở phân kỳ giữa các tia LiDAR cố định nên mật độ tia quét trên một đơn vị diện tích suy giảm tỷ lệ nghịch với bình phương khoảng cách | Xe ở khoảng cách xa (>35 m) chỉ nhận được vài đường quét thưa thớt, chỉ có vài chục điểm rời rạc | Chỉ gán nhãn khi nhìn thấy tối thiểu 15–20% thể tích hình học của vật hoặc có ảnh camera hỗ trợ; nếu quá thưa dưới ngưỡng nhận diện thì bỏ qua và đánh dấu `LOW_VISIBILITY` |
| 3. Phản xạ thấp / ít điểm bất thường | Frame s03f21 · `t4-3-phan-xa-thap.png` | Kính chắn gió trong suốt làm tia laser xuyên thấu/khúc xạ, sơn xe tối màu hấp thụ mạnh bước sóng laser làm cường độ phản xạ (intensity) giảm dưới ngưỡng thu nhận | Thân xe và kính xe bị rỗng điểm hoặc chỉ có vài điểm phản xạ yếu ở biển số, cản kim loại và mép lốp xe | Không thu nhỏ cuboid theo cụm điểm ít ỏi; phải dùng kiến thức tiên nghiệm về hình dáng xe và ảnh camera để ước lượng kích thước chuẩn của toàn bộ xe |
| 4. Điểm lạ | Frame s03f21 · `t4-4-diem-la.png` | Hiện tượng phản xạ đa đường (multipath) từ bề mặt gương/mặt đường bóng loáng, hoặc bụi/khói/nhiễu cảm biến làm sai lệch thời gian bay (ToF) | Một số điểm LiDAR lẻ loi xuất hiện ở độ sâu bất thường (dưới mặt đường z < −3 m) hoặc lơ lửng trên không trung | Tuyệt đối không gán thành vật thể; không nới rộng cuboid của các xe lân cận để nuốt các điểm lạ này |

## 5. Số từ cuboid của tôi (Bước 5)
Lấy từ `submission/D13-2A202603145-bangA.csv`; số điểm lấy từ cột `points_in` của `-tukiem.csv`. Mặt đường khoảng −1,7 m: z đáy lệch hơn 0,3 m là lơ lửng hoặc chìm.

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| s01f16, idx 1, Car | −0,988 | 1,427 | −1,702 | 5,77 | 5,86 | 270,9 | Chạm đường chuẩn (lệch 0,002 m so với −1,7 m) |
| s04f07, idx 13, Truck | −0,153 | 2,991 | −1,649 | 14,18 | 14,18 | 82,1 | Chạm đường tốt (lệch 0,051 m so với −1,7 m) |
| s04f34, idx 9, Cyclist | −0,774 | 1,374 | −1,461 | 8,75 | 8,79 | 269,7 | Chạm đường sát mép vỉa hè (mặt đường dốc/nâng nhẹ 0,24 m) |

Phân tích số liệu từ Bảng A và Phiếu tự kiểm:
(a) Cuboid gần nhất và xa nhất:
- Cuboid gần nhất: `s05f03, idx 5, Car` có `dist_m` = 3,39 m và khoảng cách không gian `r` = √(x²+y²+z²) = 3,50 m (xe con bên sườn trái).
- Cuboid xa nhất: `s04f34, idx 8, Car` có `dist_m` = 48,65 m và khoảng cách không gian `r` = 48,65 m (xe con chạy phía trước ở làn xa).

(b) Kích thước l, w, h so với Bảng 4.3:
- Xe con (`s01f16, idx 1, Car`): l = 4,71 m (chuẩn: 3,0–6,0 m), w = 1,91 m (chuẩn: 1,4–2,3 m), h = 1,43 m (chuẩn: 1,2–2,3 m). Tất cả các chiều đều nằm chuẩn xác trong khoảng guideline.
- Xe hai bánh (`s03f21, idx 14, Cyclist`): l = 2,49 m, w = 0,86 m, h = 1,55 m. So với khoảng chuẩn của Cyclist (chiều dài l: 1,0–2,4 m), chiều dài l = 2,49 m hơi vượt nhẹ khoảng chuẩn (lệch 0,09 m) do người lái chở theo giỏ đồ phía sau khiến cụm điểm kéo dài thêm.
- Một số trường hợp ngoài khoảng khác trong bảng tự kiểm: `s01f16, idx 3, Car` có h = 1,09 m (thấp hơn khoảng 1,2–2,3 m do xe sedan mui thấp và phần cản sau bị che); `s01f44, idx 2, Car` có w = 2,31 m (hơi nhỉnh hơn 2,3 m do gương chiếu hậu mở rộng tạo điểm phản xạ).

(c) Góc xoay `yaw_deg`:
- Xe cùng chiều phía trước (`s01f44, idx 1, Car`): có `yaw_deg` = 268,5°, rất gần 270° (lệch chỉ 1,5° do xe di chuyển thẳng theo làn về hướng −y).
- Xe ngược chiều (`s02f17, idx 4, Car`): có `yaw_deg` = 90,1°, rất gần 90° (hướng thẳng về +y ngược lại với hướng xe thu dữ liệu).

Mật độ theo khoảng cách (4 cuboid ở các `dist_m` khác nhau):

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| s05f03, idx 5, Car | 3,39 | 5448 |
| s01f44, idx 2, Car | 11,07 | 948 |
| s01f16, idx 5, Car | 20,10 | 301 |
| s01f44, idx 8, Car | 34,40 | 76 |

Quy luật mật độ điểm theo khoảng cách:
Mật độ điểm suy giảm xấp xỉ tỷ lệ nghịch với bình phương khoảng cách ($1/r^2$). Cụ thể: khi khoảng cách tăng gấp khoảng 3,3 lần (từ 3,39 m lên 11,07 m), số điểm giảm tới 5,7 lần (từ 5448 điểm xuống 948 điểm); khi khoảng cách tăng từ 11,07 m lên 34,40 m (tăng hơn 3,1 lần), số điểm tiếp tục giảm mạnh hơn 12,5 lần (từ 948 xuống 76 điểm). Quy luật này hoàn toàn phù hợp với đường cong phân bố thực nghiệm trong slide bài giảng (xe con trung vị: ~1968 điểm ở 0–10 m, ~718 ở 10–20 m, ~214 ở 20–30 m, ~72 ở 30–40 m).

Phần tự kiểm và điều chỉnh:
- Hai cuboid đã sửa sau khi chạy `preview` và `selfcheck`:
  1. Frame s01f16, idx 1 (Car): Ban đầu đáy hộp z_bottom đặt ở −1,95 m (bị chìm sâu vào lòng đường ~0,25 m). Sau khi kiểm tra Side view và ảnh preview chiếu lên camera, đã nâng mặt đáy lên z = −0,988 m (h = 1,427 m), đưa z_bottom về chuẩn −1,702 m khớp sát mặt đường phẳng.
  2. Frame s01f44, idx 1 (Car): Ban đầu góc xoay yaw bị ngược 180° (mũi tên hướng về phía sau xe ~88,5°). Sau khi bật Cuboid orientation và đối chiếu với ảnh camera cam01, đã xoay đảo 180° đưa yaw_deg về 268,5° chỉ đúng đầu xe cùng chiều xe chạy.
- Các cảnh báo được giữ nguyên kèm lý do:
  - Cảnh báo kích thước ngoài khoảng ở s01f16 idx 3 (h=1,09 m) và s02f17 idx 2 (length=2,34 m): Giữ nguyên vì xe bị che khuất một phần (occluded) bởi xe khác hoặc góc chụp khuất cản sau, cụm điểm thực tế chỉ thu nhận được chừng đó.
  - Cảnh báo 378/473 điểm sát ngay ngoài mép ở s01f16 idx 7 (Bus) và s03f21 idx 5 (Car): Giữ nguyên vì xe đỗ sát dải cây xanh và xe bên cạnh, các điểm ngoài mép thuộc về vật thể lân cận chứ không phải thân xe.
  - Cảnh báo đáy cao hơn mặt đường ở s04f07 idx 4, s05f44 idx 8: Giữ nguyên vì đoạn đường này mặt đường dốc lên hoặc xe nằm trên vỉa hè cao hơn lòng đường cục bộ của xe thu dữ liệu.

Đổi kiểm chéo:
- MSSV bạn ngồi cạnh đã kiểm chéo: 2A202600118.
- Hai con số được đối chiếu:
  1. Tổng số cuboid trong Bảng A (103 cuboid) và số hộp ở frame 1 (s01f16: 7 cuboid) -> Khớp hoàn toàn giữa màn hình CVAT và file CSV Bảng A.
  2. Độ cao đáy z_bottom của xe gần nhất ở s01f16 idx 1 (−1,702 m) và khoảng cách dist_m (5,77 m) -> Khớp chính xác giữa hai bên đối chiếu.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
Nhãn 3D dùng mét vì mô hình xe tự hành cần học vị trí và kích thước thực tế của vật thể trong không gian 3 chiều để tính toán khoảng cách an toàn, vận tốc và quỹ đạo điều khiển tránh va chạm, độc lập với độ phân giải và góc đặt của camera; trong khi tọa độ pixel chỉ là hình chiếu 2D thay đổi theo tiêu cự và vị trí camera. Vật thưa điểm cần quy tắc riêng vì thiếu dữ liệu hình học trực quan rất dễ dẫn đến việc vẽ sai lệch kích thước hoặc nhầm lẫn loại vật; guideline quy định chỉ gán khi nhìn thấy tối thiểu 15–20% thể tích và có bằng chứng hỗ trợ rõ ràng từ ảnh camera để đảm bảo chất lượng nhãn huấn luyện cho mô hình.

- Công cụ: Tra cứu tài liệu kỹ thuật và ChatGPT hỗ trợ định dạng bảng Markdown.
- Câu hỏi chính: Hỗ trợ rà soát cấu trúc báo cáo, kiểm tra công thức tính khoảng cách không gian 3D từ tọa độ Bảng A.
- Cách dùng kết quả: Tham khảo cấu trúc trình bày báo cáo kỹ thuật; toàn bộ dữ liệu gán nhãn, số đo trong Bảng A, phân tích hình ảnh và kiểm tra trên CVAT do tôi trực tiếp thực hiện và xác minh độc lập.
- Điều còn chưa chắc / tồn tại sau bài lab: Một số cuboid ở cự ly xa trên 40 m do số lượng điểm quá ít (dưới 20 điểm) nên chiều dài xe chủ yếu dựa theo kích thước chuẩn trung bình, có thể có sai số nhỏ về độ dài thực tế; đối với người đi xe hai bánh (Cyclist) khi di chuyển nhanh, một số điểm bị phân tán nhẹ do chuyển động làm việc căn chỉnh khung xe cần nhiều thời gian đối chiếu hơn.
