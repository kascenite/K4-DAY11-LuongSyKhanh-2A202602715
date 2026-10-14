MSSV: 202602715

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): B. Mọi bước chạy với ngân sách đầy đủ.

## 1. Phát lại log có trung thực không? (bước A)

`replay.json`: bỏ 1 log do người lái tự va chạm. Phát lại có 104 va chạm, gồm 92 `rear` và 12 `side`. Thế giới phản ứng có 1 va chạm `front`. Không kịch bản nào va chạm ở cả hai (`crash_in_both` = 0).

Planner giữ khoảng cách 1,6 s, xa hơn người lái trong log (1,1 s), và giảm tốc gần vạch bị che. Khi phát lại, xe bám đuôi TG chạy đúng quỹ đạo cũ, không biết xe ta phanh sớm hơn nên tông vào đuôi. Đó là va chạm giả, vì TG thật sẽ phanh theo.

Ví dụ `replay_only[0]`, kịch bản 9 (R-01, `shots/q1_replay_only.png`): TG tông đuôi lúc 16,6 s. Trong thế giới phản ứng không va chạm, TTC nhỏ nhất 2,71 s.

## 2. Kịch bản bất khả thi (bước B)

| Luật | Căn cứ |
|---|---|
| (π / cutDur)² · 3,5 / 2 > μg | chuyển làn quá nhanh cần lực ngang vượt ma sát lốp |
| cutBrake > μg | không xe nào phanh mạnh hơn ma sát cho phép |
| pedV > 2,5 m/s | đó là chạy, người đi bộ thường đi 1,2 đến 1,5 m/s |
| cutV ngoài (0, 22] m/s | xe đứng yên không chuyển làn được, 22 m/s quá nhanh cho phố |

`hunt.json: infeasible`: `rear_ended` 10, `cut_in` 6, `crosswalk_block` 5, `bus_bypass` 3. Cuộc săn `rear_ended` cao nhất vì điểm của nó tăng khi xe ta phải phanh gấp, nên CEM bị kéo về vùng phanh hay chuyển làn vượt μg. Đếm chúng là va chạm sẽ thổi phồng ước lượng bằng những tình huống không thể xảy ra.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k = 8, n = 3000, p̂ = 2,67e-3, Wilson 95 % [1,35e-3; 5,25e-3], cả 8 vụ là `bus_bypass`.

Để có ±20 % của p cần n ≈ 96/p ≈ 36 000 lần chạy.

`cut_in`, `crosswalk_block`, `rear_ended`, `other` chưa xuất hiện. Báo cận trên Wilson 1,28e-3 cho mỗi kiểu, không báo 0.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json`, 600 mẫu mỗi kiểu mỗi seed:

| seed | kiểu | p | hits | ess | top_share |
|---|---|---|---|---|---|
| 0 | bus_bypass | 1,13e-3 | 55 | 121 | 0,23 |
| 0 | crosswalk_block | 7,71e-6 | 78 | 146 | 0,45 |
| 1 | bus_bypass | 1,11e-3 | 49 | 115 | 0,14 |
| 1 | crosswalk_block | 6,37e-6 | 75 | 130 | 0,42 |
| 2 | bus_bypass | 8,86e-3 | 57 | 141 | 0,90 |
| 2 | crosswalk_block | 2,73e-4 | 90 | 151 | 0,98 |

`cut_in` và `rear_ended` không có hit ở cả 3 seed, báo `upper_zero_hits` = 0,025. `total`: 1,13e-3, 1,11e-3, 9,13e-3.

Seed 0 và 1 khớp nhau và thấp hơn Monte Carlo khoảng 2,4 lần, đúng như README cảnh báo. Seed 2 lệch 8 đến 40 lần vì `top_share` lên 0,90 và 0,98: một mẫu mang gần hết ước lượng, nên con số chỉ là trọng số của mẫu đó.

`budget` = 16 800 lần chạy. Với `crosswalk_block`, p ≈ 7,7e-6 và sai số tương đối 0,49, Monte Carlo cần khoảng 550 000 lần để chính xác như vậy, gấp 33 lần.

`other` lấy từ `mc.json` (0 trên 3000, cận 1,28e-3). Không săn nó vì không có điểm nguy hiểm chung để nhắm.

## 5. Các kiểu thất bại (bước C2 + 3D)

`bus_bypass` (B-01, `shots/q5_bus_bypass.png`, 13,0 s): đường ướt. Xe tím C2 bị xe buýt chắn nên lách sang làn xe ta với khe 9,4 m. Xe ta nhận ra C2 quá muộn và tông vào.

`crosswalk_block` (C-01, `shots/q5_crosswalk_block.png`, 19,5 s): người đi bộ bước ra từ sau xe buýt mà không ngó, xe ta cũng bị che. Xe ta phanh gấp trên đường ướt và dừng với thân xe nằm trên vạch.

Không tìm thấy `cut_in` và `rear_ended`: cả hai cuộc săn đều trôi về `bus_bypass` hoặc `crosswalk_block` (`hunt.json: found`). Tôi tin kết luận này ở mức thấp, vì cận trên vẫn còn 1,28e-3.

Một cuộc săn chung sẽ co về `bus_bypass`, kiểu dễ va chạm nhất, và gần như chắc chắn bỏ sót `crosswalk_block`.

## 6. Sửa planner (bước E + 3D)

Policy chọn: `cutinLat` = 3,3.

| `fix.json` | fail | fail_hi_seen | fail_hi | quãng đường, m | phanh gấp |
|---|---|---|---|---|---|
| gốc, TRAIN | 3,07e-4 | 5,72e-4 | 5,72e-4 | 211,8 | 8,1 |
| sửa, TRAIN | 3,32e-5 | 9,43e-5 | 9,43e-5 | 211,5 | 7,7 |
| sửa, TEST | 7,55e-7 | 1,59e-6 | 3,88e-2 | 213,4 | 8,0 |
| gốc, TEST | 8,54e-4 | 1,67e-3 | 2,95e-3 | 213,6 | 8,8 |

Trên TEST, bản sửa xoá hết `bus_bypass` (gốc có 21 hit) mà chỉ mất 0,1 % quãng đường. `fail_hi` của nó vẫn lớn hơn `fail` của gốc vì `bus_bypass` thành `unseen`, nhận cận 3/(0,2 · 400) = 0,0375. 400 mẫu không đủ chứng minh một kiểu đã biến mất.

Ảnh `shots/q6_fix_ghost.png` (B-01): xe đặc là policy gốc, tông C2; bóng mờ là bản sửa, đi qua an toàn.

Núm phản tác dụng: `T` = 2,0 và `cautionV` = 6,5 đều làm fail tăng khoảng 10 lần trên TRAIN (3,04e-3 và 3,01e-3). Tôi đoán xe ta để khoảng trống lớn hơn hoặc chậm lại cạnh xe buýt, nên C2 có thêm khe để chen vào.

## 7. Từ mô phỏng ra đường thật

| Giả định | Dữ liệu thật | Nếu sai |
|---|---|---|
| Khe chấp nhận khi chuyển làn | quỹ đạo từ highD, NGSIM, Waymo Open Motion | người thật chấp nhận khe nhỏ hơn thì `bus_bypass` đang bị thấp |
| Tỉ lệ người ngó qua xe buýt | video ở điểm dừng xe buýt gần vạch | ít người ngó hơn thì `crosswalk_block` đang bị thấp |
| Tỉ lệ đường ướt, μ | dữ liệu thời tiết, đo ma sát mặt đường | ướt thường hơn thì cả hai kiểu đều đang bị thấp |

## 8. Giới hạn

Chỉ có một con phố với một xe buýt và một vạch qua đường. Người lái khác đều là IDM, không ai mất tập trung hay vi phạm luật. Cảm biến hoàn hảo, trừ phần bị xe buýt che.

Thêm một tuần, tôi sẽ chạy nhiều seed hơn để xem IS hội tụ về đâu, săn lại `cut_in` và `rear_ended` bằng điểm khác, và thử logic mới ở câu 9.

## 9. Rắc rối của bạn (bước G + chương 10)

Rắc rối: người băng ngang giữa phố từ lề phải, không có vạch. Planner chỉ dự đoán người đi bộ P1 ở vạch, nên tôi đoán nó sẽ phản ứng muộn với người ở chỗ khác.

Giả định (`hazard.json`): vị trí 130 đến 300 m, tốc độ 1,0 đến 2,2 m/s, bước ra khi xe còn cách 10 đến 40 m. `exposure` = 0,02 mỗi chuyến là giả định của tôi, cần thay bằng số đếm từ dashcam.

`mine.json`, `accepted` = 0,991:

| | gốc | bản sửa |
|---|---|---|
| `avoidable` | 17 | 17 |
| `unavoidable` | 0 | 0 |
| `unattributed` | 2 | 0 |
| `low_speed` | 1 | 1 |
| P(lỗi \| rắc rối) | 1,14e-2 [7,15e-3; 1,82e-2] | như bên |
| mỗi chuyến | 2,29e-4 | như bên |

Hiếm hơn `bus_bypass` khoảng 12 lần mỗi chuyến, nhưng khi xảy ra thì planner hỏng 1,1 % số lần, và bản sửa không giúp được.

`avoidable` là phanh hết cỡ ngay lúc thấy thì tránh được, chỉ loại này tính là lỗi planner. `unattributed` là va chạm thường của thị trấn, xảy ra trước khi người băng ngang xuất hiện. Chuẩn "tránh được" lạc quan vì giả định phản ứng tức thì và phanh tối đa, nên nó đếm dư lỗi planner; nhưng nó chỉ thử phanh, không thử đánh lái, nên cũng có thể đếm thiếu.

Hình: `shots/q9_jaywalker_studio.png` (chương 10, 20,3 s, khớp `mine.json: exact`) và `shots/q9_jaywalker_lab.png` (GB-01, 19,4 s).

Tự lái: người đi bộ hiện ra lúc 17,05 s, nhưng planner vẫn tăng tốc theo xe phía trước và đến 20,0 s mới phanh. Tôi thử một người lái giả lập phanh sau 0,5 đến 1,5 s, ở mức 4 đến 7,5 m/s²: cả 9 tổ hợp đều tránh được.

Cách sửa: không núm nào sửa được, vì `pedBuffer` và `cautionV` chỉ áp dụng cho P1. Planner cần dự đoán mọi người đi bộ đang đi về phía làn mình trong khoảng 50 m phía trước, rồi nhường như với P1.
