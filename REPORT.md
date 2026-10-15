# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Hoàng Quốc Việt |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602563 |
| GitHub | Catnip-harvest |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | chia dữ liệu 4741 / seed mô hình 2561 (năm seed liên tiếp 2561–2565) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định, không đặt `--seeds`) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.277 (hạng 3) | 19.9 (hạng 3), Wilson 18.5–21.4 | BLOCK: rớt R2, R3, R4, R6 |
| IDM | 1.527 (hạng 4) | 0.14 (hạng 1), 4/2883, Wilson 0.05–0.36 | (baseline) |
| EgoMLP | 0.766 ± 0.001 (hạng 2) | 14.8–32.3 qua 5 seed, trung bình 23.0 (hạng 4) | BLOCK: rớt R2, R3, R4, R5, R6 |
| PercMLP | 0.623 ± 0.002 (hạng 1) | 1.3–1.7 qua 5 seed, trung bình 1.5 (hạng 2) | BLOCK: rớt R2, R3, R4, R6 |
| PercMLP+ (sau khi sửa) | 0.813 ± 0.013 | 0.28–0.35 qua 5 seed, trung bình 0.30 | RELEASE: qua cả 7 quy tắc |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Có ba điều tôi phải biết trước khi tin con số nào. Thứ nhất, cột `Acc_LV` hỏng: tương quan của nó với đạo hàm tốc độ xe dẫn chỉ 0.045 (trong khi `Acc_FAV` của xe theo sau là 0.929), nên tôi không dùng nó mà tự tính gia tốc xe dẫn từ `Speed_LV` đã làm mượt (trung bình trượt 5 khung rồi lấy sai phân trung tâm ở 10 Hz); chính đại lượng này dùng để khai thác ca phanh.
Thứ hai, có 383 quỹ đạo trùng hệt nhau nằm ở các id xa nhau: 9993 quỹ đạo thô còn 9610 sau khi loại trùng. Nếu để nguyên, một bản sao nằm ở train và bản kia ở test thì mô hình "thuộc bài", điểm đẹp giả.
Thứ ba, log gần như không có phanh mạnh: gia tốc xe dẫn đã làm mượt ở phân vị 0.1 chỉ là -2.77 m/s², phân vị 1 là -1.95 m/s², nên ngưỡng -2.0 chỉ bắt phần đuôi (khoảng cách thì đáng tin: `Spatial_Gap` = Pos_LV - Pos_FAV - 4.5 m, sai lệch tối đa 3.6e-11 m).
Tôi chia theo quỹ đạo (seed chia 4741, 30 % làm test = 2883 quỹ đạo) chứ không theo khung hình vì các khung kề nhau của một quỹ đạo gần như giống hệt: chia theo khung thì test chứa gần đúng những gì train đã thấy, và số đo vòng kín (lái cả quỹ đạo) cũng không còn ý nghĩa.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 (đổi thứ tự phép tính) khác bit ở 600/600 đoạn nhưng vượt dung sai ở 0/600 đoạn: lệch lớn nhất chỉ 1.2e-4 m/s², nhỏ hơn atol 1e-3 + rtol 1e-3·|lệnh cũ|; còn chạy lại đúng cùng mã thì 0 đoạn khác, 0 đoạn khác bit. Phép cộng float không có tính kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001 còn 0.1+(0.2+0.3) = 0.6; tổng 50 nghìn số float32 xuôi và ngược lệch nhau 6.9e-6 tương đối), nên so bằng `==` sẽ báo hỏng oan ở mọi đoạn, dung sai mới báo đúng thứ cần biết.
Thêm AEB (phanh -6 m/s² khi TTC < 1.5 s) là đổi hành vi thật: 14/600 đoạn lệch vượt dung sai (15 đoạn không giống bit), lệch tối đa 1.84 m/s². Tôi mở đoạn 91: chỉ 1 khung trong 83 khung lệch (khung 82: gap 6.16 m, tiến gần 4.2 m/s, TTC 1.47 s, lệnh IDM -4.56 → AEB -6.0); đoạn 145 có 7 khung lệch bắt đầu từ khung 34, các khung đầu có TTC 1.42–1.49 s và lệnh đổi từ khoảng -5.3 đến -5.7 sang -6.0.
Lệch đúng ở khung TTC thấp vì luật AEB chỉ chạm tới lệnh khi TTC < 1.5 s và IDM tự nó phanh nhẹ hơn -6; ở mọi khung còn lại hai bản ra cùng một lệnh.
Phát nhanh nhất không có hạn chót nên 0 khung muộn, 0 khung rớt trên 59722 khung. Phát theo thời gian thật (chi phí tính 6 ms, 1 % khung tốn 140 ms vượt hạn chót 100 ms) có 582 khung muộn và 575 khung rớt, lệnh cũ bị giữ lại, kéo theo 376/600 đoạn đổi lệnh vượt dung sai (lệch tới 7.40 m/s²): phát nhanh nhất che mất lỗi trễ hạn mà xe thật sẽ gặp.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của tôi: tốc độ cruise 14.0 m/s (suy ra từ MSSV), khoảng cách 2.76–59.3 m (phân vị 2–98 của log) × tốc độ tiến gần 0–14 m/s, 60 × 45 = 2700 ô. Biên bất khả thi là gap < dv·t_react + dv²/(2·9): có 3.6 % ô không ai tránh được khi t_react = 0 và 10.2 % khi t_react = 0.6 s. Kiểm chứng bằng xe phanh hết cỡ: 0 ô lệch ngoài vùng biên ở cả hai độ trễ (0/2682 và 0/2662 ô), nên `unavoidable` và bộ mô phỏng khớp nhau.
Khi độ trễ đi từ 0 lên 0.6 s, vùng đâm của IDM tăng từ 3.6 % lên 10.2 % (thêm 6.6 điểm phần trăm), nhưng phần "đâm mà lẽ ra tránh được" của IDM vẫn là 0.0 % ở cả hai độ trễ: IDM chỉ đâm đúng nơi không ai tránh nổi. Constant-velocity thì đâm tới 77.0 % (t_react 0) và 70.4 % (0.6 s) lưới ở phần tránh được. Đây là cách phân biệt "ai cũng đâm" với "bộ điều khiển đâm".
Log không phủ vùng khó: người lái trong log phanh mạnh nhất chỉ 3.6 m/s² (phân vị 99.9) trong khi cơ cấu phanh làm được 9 m/s², và 12.7 % lưới đòi phanh mạnh hơn 3.6. Chỉ khai thác được 614 ca phanh (6.4 % quỹ đạo) với giảm tốc đỉnh trung vị -2.5 và phân vị 99 là -4.2 m/s²; trên hình `scenarios_idm.png` các chấm trắng dồn ở góc dưới (tốc độ tiến gần dưới khoảng 4–6 m/s), gần như không có chấm nào ở vùng tiến gần nhanh và gap nhỏ.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng đầy đủ nằm ở mục "Kết quả cuối". Theo L2 vòng hở: PercMLP 0.623 m (hạng 1), EgoMLP 0.766 (hạng 2), constant-velocity 1.277 (hạng 3), IDM 1.527 (hạng 4); theo va chạm vòng kín trung bình trên 2883 log: IDM 0.14 % (hạng 1), PercMLP 1.5 % (hạng 2), constant-velocity 19.9 % (hạng 3), EgoMLP 23.0 % (hạng 4). Bốn cặp đảo thứ tự: const-vel/IDM, const-vel/EgoMLP, IDM/EgoMLP và IDM/PercMLP; IDM từ chót bảng L2 nhảy lên đầu bảng va chạm.
IDM có L2 xấu (1.527 m, kém cả giữ tốc độ hằng 1.277 m) vì L2 chỉ đo mức bắt chước người lái trong 3 s: IDM là luật viết tay với T = 1.2 s, còn người lái trong log giữ khoảng cách thời gian trung vị 1.86 s, nên dự báo vị trí lệch người. Nhưng nó luôn phản ứng đúng với khoảng cách và tốc độ tiến gần nên khi tự lái chỉ đâm 4/2883 log; vòng hở không phạt "khác người mà an toàn".
EgoMLP có L2 tốt hơn constant-velocity (0.766 so với 1.277) vì nó học được thói quen tăng giảm tốc từ 2 s lịch sử tốc độ của chính nó, nhưng nó không thấy khoảng cách và tốc độ xe dẫn: xe trước phanh là điều không thể suy ra từ tốc độ của mình, nên nó đâm 14.8–32.3 % qua 5 seed và tới 61.3 % trên các log có xe dẫn phanh (seed xấu nhất).
Vòng hở che lỗi vì lát cắt "xe dẫn sắp phanh" chỉ chiếm 4.4 % số cửa sổ test (L2 trên lát cắt của EgoMLP là 1.098 so với 0.766 chung), và vì mỗi cửa sổ được chấm riêng trên lịch sử thật của người lái, còn vòng kín để sai số tự tích luỹ. PercMLP nhìn thấy gap nên đâm ít hơn hẳn (1.3–1.7 %) nhưng vẫn cao hơn IDM 0.14 %.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động rất mạnh giữa 5 seed huấn luyện (2561–2565): 32.3, 14.8, 28.5, 16.7 và 22.9 % va chạm trên cùng 2883 log, tức dải 14.8–32.3 % (rộng 17.5 điểm phần trăm), trong khi L2 vòng hở của nó hầu như không đổi (0.766 ± 0.001), nên điểm vòng hở không báo trước sự bất ổn này; R5 (dải ≤ 1 điểm) vì thế rớt.
Khoảng tin cậy Wilson 95 % của từng seed chỉ rộng cỡ ±1.3 đến ±1.7 điểm: seed xấu nhất 32.3 % có [30.6, 34.1], seed tốt nhất 14.8 % có [13.6, 16.2], hai khoảng không chồng nhau. Wilson chỉ đo sai số lấy mẫu của 2883 quỹ đạo cho MỘT mô hình đã huấn luyện, không đo độ ngẫu nhiên của việc huấn luyện; chỉ dải qua seed mới nói được điều sau.
Nếu chỉ chạy một seed, seed 2562 cho 14.8 % [13.6, 16.2], tức "tốt hơn constant-velocity" (19.9 %, [18.5, 21.4]); còn seed 2561 cho 32.3 %, tệ hơn constant-velocity nhiều. Cả hai kết luận đều có khoảng tin cậy hẹp, trông rất chắc, mà ngược nhau. Báo một lần chạy là rút ngẫu nhiên một điểm trong dải rồi gán cho nó độ chắc chắn của khoảng tin cậy, nên quyết định phát hành sẽ đổi theo hạt giống chứ không theo mô hình.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

T hiệu chỉnh là 1.4 s (xe sau IDM, v0 = 20 m/s, trễ phản ứng 0.6 s, đặt cách ego 21.6 m). Khoảng cách Wasserstein-1 giữa phân bố thời khoảng cách của xe IDM này và của người lái trong log là 0.36 (T = 1.2 s cũng 0.36, T = 2.4 s là 0.88), còn một xe không bao giờ phản ứng là 0.82: xe sau mô phỏng gần người lái hơn hẳn (người lái trong log có thời khoảng cách trung vị 1.86 s).
Tỷ lệ ô lưới cắt vào có va chạm bất kỳ (trước hoặc sau), t_react = 0.6 s, xe sau phát lại → phản ứng: const-vel 80.6 → 80.6 %; IDM 80.9 → 10.2 %; EgoMLP 80.7 → 80.7 %; PercMLP 76.4 → 32.4 %; PercMLP+ 81.2 → 10.7 %. Riêng va chạm phía sau: IDM 70.6 → 0.0 %, PercMLP 44.1 → 0.0 %, PercMLP+ 71.0 → 0.0 %; ở t_react = 0 thì IDM 80.3 → 3.6 %.
"Va chạm giả" vì xe sau phát lại cứ chạy đúng tốc độ trong log bất kể ego làm gì: khi ego phanh gấp để tránh xe cắt vào, xe sau đâm vào đuôi, tức IDM bị phạt vì phanh đúng, và đó không phải lỗi của nó. Const-vel và EgoMLP không phanh nên không có va chạm phía sau (0.0 %), họ chỉ đâm phía trước.
Bỏ tác nhân phản ứng làm đổi kết luận: trong thế giới phát lại IDM (80.9 %) trông tệ ngang const-vel (80.6 %) và không hơn gì PercMLP (76.4 %); khi xe sau phản ứng IDM chỉ còn 10.2 %, đúng bằng phần bất khả thi 10.2 % của lưới, tốt hơn PercMLP 32.4 % và hơn hẳn const-vel 80.6 %.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

R2, R3, R4, R6 chặn PercMLP (R1, R5, R7 qua). R2 trên log: seed xấu nhất 1.7 % (n=2883, cận trên Wilson 2.2 %) > ngưỡng 1.1 % = IDM 0.1 % + 1 điểm; R3 lát cắt xe dẫn phanh: 6.1 % (cận trên 10.6 %) > 1.1 % + 5 điểm; R4 lưới cắt vào (t_react 0.6): 22.1 % ô tránh được mà vẫn đâm > 0.0 % + 2 điểm; R6 xe sau phản ứng: 32.4 % > IDM 10.2 % + 2 điểm.
Có hai nguyên nhân gốc. (a) Kịch bản mà log không chứa: người lái trong log phanh tối đa 3.6 m/s² và 12.7 % lưới đòi hơn, nên PercMLP chưa từng học các cú cắt vào gấp (R4, R6 rớt; ở cruise 20 m/s nó đâm 23.0–43.3 % lưới tránh được). (b) Sai số tích luỹ khi tự lái (R2, R3): trên log nó đâm 1.3–1.7 % so với IDM 0.14 % dù L2 vòng hở tốt nhất (0.623 m), vì lệch nhỏ mỗi bước đẩy xe vào trạng thái người lái chưa từng ở rồi cộng dồn.
Tôi chữa (a) bằng 20000 kịch bản cắt vào mô phỏng do IDM gắn nhãn, và chữa (b) bằng 2 vòng DAgger: cho mô hình tự lái trên log huấn luyện, ghi lại các trạng thái nó tự đi vào, gắn nhãn bằng dự báo 3 s của IDM ở đúng các trạng thái đó rồi huấn luyện lại. PercMLP+ (5 seed) đạt 0.28–0.35 % trên log (cận trên 0.6 % ≤ 1.1 %), lát cắt phanh 1.7 % (cận trên 4.8 %), lưới 0.5 % ≤ 2.0 %, xe sau phản ứng 10.7 % ≤ 12.2 %, qua cả 7 quy tắc nên cổng báo RELEASE.
Nhưng bằng chứng chỉ cho thấy "không kém hơn", không phải "tốt hơn" IDM: IDM 0.14 % (4/2883, [0.05, 0.36]) và PercMLP+ seed xấu nhất 0.35 % (10/2883, [0.19, 0.64]) chồng nhau, xe sau phản ứng 10.7 so với 10.2 %. Nó học nhãn của IDM nên khó vượt thầy, L2 vòng hở của nó còn tệ đi (0.623 → 0.813 m), và ngoài miền huấn luyện (cruise 20 m/s, huấn luyện ≤ 16) nó vẫn đâm 0.5–1.7 % lưới tránh được trong khi IDM là 0.0 %.
Vì vậy tôi đề xuất chưa thay IDM: chạy bóng PercMLP+ song song IDM trên xe thật, mở rộng kịch bản (tốc độ cruise ngoài 4–16 m/s, nhiễu cảm biến) và thêm log trước khi quyết định.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Giới hạn 1, thế giới một chiều và không cảm biến: chỉ có điều khiển dọc, planner nhận gap và tốc độ xe dẫn chính xác, tôi chỉ thử một mức nhiễu (gap ±1 m, tốc độ ±0.5 m/s: PercMLP 1.2–1.8 %, PercMLP+ 0.28–0.38 %), không có che khuất, trễ cảm biến hay đường cong, nên con số 0.28–0.35 % của PercMLP+ chỉ đúng cho thế giới đơn giản đó.
Giới hạn 2, xe dẫn trong log không phản ứng với xe thử và xe sau chỉ là IDM T = 1.4 s do chính chúng ta hiệu chỉnh: bảng xe sau phản ứng (IDM 10.2 %, PercMLP+ 10.7 %) phụ thuộc giả định đó, và khoảng cách giữa thế giới phát lại (IDM 80.9 %) và thế giới phản ứng (10.2 %) cho thấy kết luận nhạy với giả định đến mức nào.
Giới hạn 3, dữ liệu và kịch bản hẹp: Argoverse nội đô với xe dẫn trung vị 8.1 m/s, p99 15.6 m/s; người lái phanh tối đa 3.6 m/s²; chỉ 614 ca phanh (6.4 %) và lát cắt phanh test chỉ 181 quỹ đạo nên cận trên R3 của PercMLP+ vẫn tới 4.8 % cho tỷ lệ 1.7 %; họ cắt vào do chúng ta tự chọn tham số, và ở cruise 20 m/s ngoài miền huấn luyện PercMLP+ đã đâm 0.5–1.7 % lưới tránh được.
Giới hạn 4, ngưỡng cổng (1, 5, 2 điểm) là lựa chọn của đội và 2883 log không đủ để tách 0.14 % khỏi 0.35 % (khoảng tin cậy chồng nhau); va chạm ở đây chỉ là gap ≤ 0, không cho biết mức nghiêm trọng. Để thu hẹp khoảng cách: chạy bóng trên xe thật, thêm dữ liệu cao tốc, mưa, đêm, kịch bản đối kháng và hardware-in-the-loop, nhiều seed hơn, rồi cho qua lại cùng cổng.
