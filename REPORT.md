# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Pham Minh Hieu` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602919` |
| GitHub | `BenKei-19` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `3761 / 8259 (dùng 8259..8263)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.254 | 19.0 (549/2883) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.528 | 0.14 (4/2883) | (baseline) |
| EgoMLP | 0.761 ± 0.002 (0.758..0.764) | 16.6 .. 26.8 (TB 20.5) | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.619 ± 0.002 | 0.5 .. 1.2 (TB 0.85) | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.811 ± 0.017 | 0.3 .. 0.7 (TB 0.44) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ba vấn đề trong `data_audit.json`. **(1) `Acc_LV` hỏng:** tương quan với đạo hàm của `Speed_LV` chỉ **0.045**, trong khi cột tương ứng của xe sau `Acc_FAV` đạt **0.929**. Cột này gần như là nhiễu, nếu dùng nó để khai thác ca phanh thì sẽ bắt được ca giả và bỏ sót ca thật. Mình dùng `a_lead`: lấy `Speed_LV` làm mượt (trung bình trượt 5 mẫu) rồi lấy sai phân trung tâm. Với gia tốc này, phân vị 0.1 % là −2.77 m/s², p1 là −1.95 m/s². **(2) Quỹ đạo trùng hệt:** **383** trong **9.993** quỹ đạo là bản sao, nên còn lại **9.610**. Các bản sao nằm ở ID xa nhau, nên một bản có thể rơi vào train và bản kia vào test. Khi đó điểm test là điểm trên dữ liệu đã học, tức là bị rò rỉ. **(3) Chia theo quỹ đạo, không theo khung:** log 10 Hz nên hai khung liền nhau gần như giống nhau. Chia theo khung thì test chứa "hàng xóm" của train, điểm sẽ đẹp giả. Mình chia 70/30 theo quỹ đạo (split seed 3761), được **2.883** quỹ đạo test. Ngoài ra `Spatial_Gap` khớp đúng `Pos_LV − Pos_FAV − 4.5 m` (lệch tối đa 3.6e-11 m), nên cột khoảng cách tin được.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác, 0 đoạn khác bit. Bản refactor float32 khác bit ở **600/600** đoạn nhưng **0/600** vượt dung sai (atol = rtol = 1e-3 m/s²), lệch lớn nhất **1.2e-4 m/s²**. Phép cộng số thực không kết hợp: (0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6, và cộng 50k số float32 xuôi hay ngược cho 42.32095 so với 42.32066. Vì vậy so từng bit là phép thử sai. Dung sai 1e-3 m/s² lớn gấp khoảng 8 lần nhiễu số học, nhưng vẫn quá nhỏ để có ý nghĩa vật lý cho lệnh gia tốc. Thêm AEB làm lệch **7/600** đoạn (78, 135, 194, 299, 307, ...). Mở đoạn 135: chỉ các khung 82–85 lệch. Ở đó khoảng cách giảm từ 5.11 xuống 4.24 m, tốc độ tiến gần 3.42→3.12 m/s, nên **TTC = 1.49→1.36 s < 1.5 s**: AEB kích hoạt và lệnh đổi từ −4.44 thành −6.0 m/s². Đoạn 78 chỉ lệch ở khung 82 (TTC 1.47 s), đoạn 194 ở khung 76–77 (TTC 1.40/1.48 s). Ở mọi khung khác `min(cmd, −6)` không đổi gì. Phát lại là vòng hở (log không phản ứng với lệnh mới) nên chỗ lệch không lan sang khung sau. Về chế độ phát: phát nhanh nhất có **0 khung trễ, 0 khung rớt**, vì thông điệp sau chỉ được gửi khi thông điệp trước xử lý xong, tức là không có đồng hồ và không có hạn chót. Phát theo thời gian thật lộ ra **584 khung trễ + 575 khung rớt / 59.921** khung, do các đợt xử lý 140 ms vượt hạn 100 ms. Kết quả là 369/600 đoạn đổi lệnh, lệch tới 4.07 m/s², vì bộ điều khiển phải giữ lệnh cũ, có khi đúng lúc cần phanh.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của mình: cruise **14 m/s**, khoảng cách 2.76–59.31 m (p2–p98 của log), tốc độ tiến gần 0–14 m/s, **60 × 45 = 2.700 ô**. Ô bất khả thi (phanh 9 m/s² sau thời gian trễ vẫn đâm) chiếm **3.6 %** ở t_react = 0 và **10.2 %** ở 0.6 s. Bộ mô phỏng khớp biên vật lý: bộ phanh hết cỡ có **0 ô lệch** xa biên ở cả hai độ trễ. Vùng đâm của IDM tăng từ **3.6 % lên 10.2 %** (+6.6 điểm), đúng bằng vùng bất khả thi, nên phần IDM "tự gây ra" là **0.0 %** ở cả hai độ trễ: mọi ca IDM đâm đều là ca ai cũng đâm. So với constant-velocity: đâm 80.6 %, trong đó 77.0 % (t=0) và 70.4 % (t=0.6) là tránh được. Nếu chỉ đếm va chạm thì IDM ở 0.6 s trông tệ hơn hẳn ở 0 s, dù nó không làm sai gì. Về độ phủ: khai thác được **614** ca xe dẫn phanh / 9.610 quỹ đạo (6.4 %), giảm tốc đỉnh p50 −2.5, p99 −4.2 m/s². Người lái trong log phanh mạnh nhất khoảng **3.6 m/s²**, trong khi cơ cấu chấp hành làm được 9 m/s², và **13 %** ô của lưới đòi phanh mạnh hơn mức đó. Trên `scenarios_idm.png`, các chấm trắng tập trung ở tốc độ tiến gần dưới ~4 m/s. Vùng TTC thấp, sát biên đâm, gần như không có ca nào: đó chính là vùng log không dạy được cho mô hình học.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|
| const-vel | 1.254 | 3 | 19.0 % | 3 |
| IDM | 1.528 | 4 | 0.14 % | 1 |
| EgoMLP | 0.761 | 2 | 20.5 % | 4 |
| PercMLP | 0.619 | 1 | 0.85 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). **IDM** có L2 tệ nhất (1.528 m) vì nó không bắt chước người: nó có thời khoảng cách riêng T = 1.2 s (người trong log giữ trung vị 1.87 s) và dự báo với giả định xe dẫn giữ nguyên tốc độ. L2 đo độ *giống người*, không đo độ *an toàn*. Khi lái vòng kín, số hạng (s\*/gap)² của IDM tăng mạnh khi khoảng cách co lại, nên nó phanh theo khoảng cách và tốc độ tiến gần đang thấy. Đó là phản hồi, nên sai số không tích luỹ, và va chạm chỉ 4/2883. **EgoMLP** thắng const-vel về L2 (0.761 so với 1.254) vì trên log, lịch sử tốc độ của chính xe đã chứa phản ứng của *người* với xe dẫn, nên ngoại suy xu hướng là đoán đúng. Khi tự lái, lịch sử đó do chính nó tạo ra và không còn thông tin về xe dẫn. Nó không thấy khoảng cách hay tốc độ xe dẫn, nên không biết khi nào phải phanh: đâm 16.6–26.8 %, trên lát xe dẫn phanh tới 42.3 %. L2 trung bình cũng che lát cắt khó: lát "xe dẫn sắp phanh" chỉ chiếm 4.5 % cửa sổ, và L2 trên lát đó luôn cao hơn (PercMLP 0.858 so với 0.619).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed, EgoMLP đâm từ **16.6 %** (seed 8263, 479/2883, CI95 [15.3, 18.0]) đến **26.8 %** (seed 8262, 774/2883, CI95 [25.3, 28.5]), dải **10.2 điểm**. Trong khi đó L2 vòng hở gần như không đổi (0.758–0.764 m), tức là vòng hở không thấy sự bất ổn này. Khoảng Wilson của từng seed chỉ rộng khoảng ±1.5 điểm và hai khoảng của seed tốt nhất và xấu nhất **không chồng nhau**. Vậy khác biệt giữa các seed là thật, do quá trình huấn luyện, không phải nhiễu lấy mẫu trên 2.883 quỹ đạo test. Khoảng Wilson chỉ đo độ bất định của tập test với *một* mô hình, không đo độ bất định của quy trình huấn luyện. Nếu chỉ chạy một seed, mình có thể báo "EgoMLP 16.6 %, tốt hơn constant-velocity 19.0 %", hoặc "26.8 %, tệ hơn hẳn": hai kết luận ngược nhau, cùng đúng thống kê. Với PercMLP còn nguy hiểm hơn: seed 8261 có 0.49 % (CI [0.29, 0.81]), cận trên 0.81 % ≤ 0.14 % + 1 điểm, nên **qua R2** nếu chỉ chạy seed đó. Seed tệ nhất 8262 có 1.18 % (cận trên 1.64 %) và bị chặn. Vì thế cổng dùng **seed tệ nhất** và R5 đòi dải ≤ 1 điểm (EgoMLP rớt R5).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ ô của lưới cắt vào có va chạm bất kỳ (trước hoặc sau), t_react = 0.6 s (`agents.json`):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 80.6 % | 80.6 % | 0.0 % / 0.0 % |
| IDM | 80.9 % | 10.2 % | 70.6 % / 0.0 % |
| EgoMLP | 83.3 % | 83.3 % | 0.0 % / 0.0 % |
| PercMLP | 79.9 % | 30.6 % | 50.2 % / 0.0 % |

Người lái trong log giữ thời khoảng cách trung vị **1.87 s**. Hiệu chỉnh trên lưới T cho **T = 1.4 s**, với khoảng cách Wasserstein-1 giữa hai phân bố thời khoảng cách là **0.35**. Một xe không bao giờ phản ứng có W1 **0.87**, nên xe sau IDM chân thực hơn rõ rệt, dù vẫn chưa khớp hẳn. Khi xe sau chỉ *phát lại*, nó giữ tốc độ trong log dù xe thử phanh. IDM phanh đúng lúc cho xe cắt vào nên bị húc đuôi ở **70.6 %** lưới. Đây là va chạm giả: người lái thật phía sau sẽ phanh theo. Bị tính lỗi này, IDM (80.9 %) trông **tệ hơn** const-vel (80.6 %), vì const-vel không bao giờ phanh nên không bị húc đuôi. Ở t_react = 0.0, PercMLP (82.4 %) cũng trông ngang IDM (80.3 %). Khi xe sau phản ứng, va chạm phía sau về **0 %** và IDM còn **10.2 %**, đúng bằng vùng bất khả thi, còn PercMLP còn 30.6 %. Kết luận đảo ngược: phạt bộ điều khiển phanh đúng là sai do thế giới mô phỏng, không phải do bộ điều khiển.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 4 quy tắc, có hai nguyên nhân gốc. **(a) Kịch bản log không chứa → R4, R6.** Trên lưới cắt vào (0.6 s), PercMLP đâm ở **20.3 %** ô tránh được, so với IDM 0.0 % + 2 điểm. Với xe sau phản ứng là 30.6 % so với 10.2 % + 2. Đây đúng là vùng tiến gần nhanh mà mục câu 3 cho thấy log không phủ (13 % lưới đòi phanh mạnh hơn 3.6 m/s² của người). **(b) Sai số tích luỹ khi tự lái → R2, R3.** L2 vòng hở của nó tốt nhất (0.619 m), nhưng vòng kín seed tệ nhất có **1.2 %** (34/2883, cận trên 1.6 % > 0.14 % + 1). Lát xe dẫn phanh: 2.9 %, cận trên 6.5 % > 1.1 % + 5. Lỗi nhỏ mỗi bước đẩy xe vào các trạng thái người chưa từng đi qua. Mình sửa bằng hai cách. Cách 1: thêm cửa sổ từ **20.000** ca cắt vào mô phỏng, gắn nhãn bằng IDM, để chữa (a). Cách 2: **DAgger** 2 vòng, gắn nhãn IDM ở chính các trạng thái mô hình tự đi vào, để chữa (b). Sau khi sửa: R4 **20.3 → 0.3 %**, R6 30.6 → 10.6 %, R2 seed tệ nhất 0.73 % (cận trên **1.1 %**, ngưỡng là 1.14 %, qua sát nút), R3 1.7 % (cận trên 4.9 %). Cả 7 quy tắc qua → RELEASE. **Nhưng PercMLP+ không tốt hơn IDM.** Trên log nó đâm 0.35–0.73 % (TB 0.44 %), so với IDM 0.14 %, cao hơn ở *mọi* seed. Lưới: 0.3 % so với 0.0 %. Ở tốc độ cruise chưa thấy khi huấn luyện (20 m/s): 0.4–2.5 % so với 0.0 %. Nó học từ chính IDM nên giỏi nhất cũng chỉ bằng thầy. Cổng chỉ chứng minh "không kém hơn IDM quá biên", không chứng minh "tốt hơn". L2 vòng hở còn *xấu đi* (0.619 → 0.811 m) trong khi vòng kín tốt lên: điểm vòng hở không đo an toàn. Đề xuất: **chưa thay IDM**. Trước hết cần trả lời PercMLP+ mang lại lợi ích gì (êm hơn, giống người hơn?). Sau đó chạy shadow mode bên cạnh IDM, thêm seed và quỹ đạo test để thu hẹp CI, kiểm thử trên họ kịch bản *không dùng để huấn luyện* (tốc độ cao, có nhiễu cảm biến và độ trễ), rồi mới quyết định.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

**(1) Kiểm thử trên đúng họ kịch bản đã dùng để sửa.** `cutin_windows` sinh ca cắt vào với tốc độ 4–16 m/s, bằng cùng bộ mô phỏng và cùng IDM làm nhãn. Lưới của R4 lại dùng cruise 14 m/s, nằm trong khoảng đó. Vì vậy R4 = 0.3 % một phần là điểm trên dữ liệu đã học. Ngoài khoảng đó (20 m/s), PercMLP+ đâm 0.4–2.5 % lưới trong khi IDM 0.0 %. Cổng có thể đang cho qua một mô hình chỉ thuộc bài. **(2) Thế giới mô phỏng là giả định của ta, và nó nghiêng về IDM.** Bài lab chỉ có một chiều dọc; xe dẫn trong log không phản ứng với xe thử; cắt vào là một vật xuất hiện tức thời; xe sau phản ứng cũng là IDM, hiệu chỉnh chỉ bằng thời khoảng cách (W1 0.35, chưa khớp hẳn). Baseline, thầy dạy và thế giới đều cùng họ IDM, nên kết luận "không kém IDM" có thể không còn đúng với người lái thật hay khi có chuyển làn. **(3) Không có cảm biến và thời gian thực.** Vòng kín trên log dùng nhận thức hoàn hảo, độ trễ 0. Thử nhiễu chỉ là Gauss (±1 m, ±0.5 m/s), và PercMLP+ đã lên 0.38–0.62 % so với IDM 0.17 %. Bước 1 cho thấy phát thời gian thật có 584 khung trễ và 575 khung rớt. Không có phát hiện sai, bỏ sót hay che khuất. **(4) Thống kê sự kiện hiếm và miền dữ liệu hẹp.** Chỉ 2.883 quỹ đạo test, 175 có xe dẫn phanh. Cận trên của IDM đã là 0.36 %, nên tập này không phân biệt được 0.1 % với 0.3 %, trong khi an toàn thật cần tỷ lệ nhỏ hơn nhiều bậc. Dữ liệu chỉ là Argoverse 2 đô thị, đoạn 7–11 s, tốc độ xe dẫn p99 khoảng 15.6 m/s. **Việc tiếp theo:** thử trên họ kịch bản tách riêng khỏi huấn luyện (tốc độ, kiểu cắt vào, xe dẫn phanh gắt); dùng tác nhân phản ứng học từ dữ liệu, mô phỏng nhiều làn; mô hình nhiễu cảm biến thật và chạy thời gian thật hoặc hardware-in-the-loop; dùng nhiều dữ liệu và seed hơn để CI đủ hẹp; cuối cùng là shadow mode trên xe trước khi giao quyền điều khiển.
