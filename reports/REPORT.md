# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: NGUYỄN TRỌNG HÙNG - 2A202602057

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Tập pool chưa gán nhãn và tập test được chia theo trục thời gian với vùng đệm vì camera đứng yên và các xe di chuyển liên tục trên cùng tuyến đường. Theo `data/DATA.md`, video 160 giây được lấy mẫu 2.5 fps, sau đó tách thành 20 ảnh test ở các đoạn thời gian quanh 20s, 60s, 100s và 140s, còn 112 ảnh nằm ở vùng đệm 4 giây trước/sau mỗi đoạn test và 268 ảnh còn lại là pool. Các ảnh ở pool gần test nhất vẫn cách test khoảng 4.4 giây, nên cùng một xe không nằm đồng thời trong huấn luyện và kiểm thử ở khoảng thời gian quá gần. Nếu chia ngẫu nhiên, một chiếc xe sẽ xuất hiện ở cả train và test, dẫn đến rò rỉ dữ liệu (data leakage): mô hình sẽ được chấm trên những xe nó đã từng thấy, làm AP50, precision và recall cao giả tạo, không phản ánh độ tổng quát thực sự. Đây là lý do giải thích vì sao lab dùng split theo thời gian thay vì random split.
2A202602102

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 trong `reports/rounds_table.md` là:

| vòng | model                                   | ảnh train | box train |  AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 |    F1 | R small | R medium | R large |
| ---: | --------------------------------------- | --------: | --------: | ----: | -------------------: | -----: | -----: | ----: | ------: | -------: | ------: |
|    0 | yolov8n cold start (COCO car+bus+truck) |         0 |         0 | 0.771 |                    — |  0.925 |  0.489 | 0.640 |   0.182 |    0.547 |   0.561 |

Dựa vào `outputs/compare_round0.jpg`, mô hình cold start hoạt động khá tốt trên các xe lớn/đủ sáng ở giữa khung hình nhưng vẫn bỏ sót nhiều xe nhỏ, xe tối, xe ở mép ảnh hoặc xe bị cắt một phần bởi đường phân cách hoặc mép khung. Đây là pattern rõ nhất ở recall theo kích thước: recall nhỏ chỉ 0.182, medium 0.547, large 0.561. Như vậy, độ yếu lớn nhất nằm ở xe nhỏ và xe bị che/không đủ thông tin tương phản. Một trường hợp cần kiểm lại nhãn tham chiếu trước khi kết luận mô hình sai là `frame_0392.jpg`: trong blind scan tôi đếm 26 xe, và trong `REVIEW_LOG.csv` có các sửa đổi như bỏ 2 box nhận nhầm xe buýt lớn, giữ và chỉnh 4 box xe con mép dưới/mép phải. Điều này cho thấy không phải mọi “bỏ sót” đều là lỗi của mô hình; có những vật thể ở sát biên hoặc quá tối, và cả nhãn tham chiếu ban đầu cũng cần được rà lại để tránh kết luận sai.

## 3. Chiến lược chọn mẫu

Chúng ta dùng hàm điểm:

score = W_U·U + W_A·A + W_D·D

với W_U = 0.5, W_A = 0.3, W_D = 0.2. Ở đây:

- U là độ bất định trung bình của 5 box khó nhất trong một ảnh; ảnh nào model phân vân nhất thì U cao.
- A là tỉ lệ box “mập mờ” trong khoảng xác suất 0.15 ≤ c < 0.50, chuẩn hóa theo max trong pool.
- D là độ đa dạng thời gian, đo khoảng cách tới frame đã gán gần nhất; `MIN_GAP_S = 2.0` giữ cho các frame được chọn không gần nhau quá mức, tránh chọn ảnh gần trùng lặp.

Điểm này hướng chọn những ảnh có nhiều xe mơ hồ, nhiều box khó, và không lặp quá nhiều cảnh tương tự. Trong `reports/SELECTION.md`, ba frame trọng tâm là `frame_0182.jpg` (score 0.9591), `frame_0369.jpg` (0.9324) và `frame_0380.jpg` (0.9170). Ba ảnh này đều nằm trong top đầu CSV và có số box lớn cũng như số trường hợp ambiguous cao (`n_ambiguous` 18, 16, 15). Đây là những frame mà model không chắc chắn và có nhiều xe bị bỏ sót/đo nhầm, nên rất đáng để sửa nhãn để fine-tune. Một frame khác có điểm cao nhưng không được chọn là `frame_0372.jpg` (score 0.9101) — rank 6 — không chọn. Frame này nằm rất gần frame_0369.jpg về thời gian. Do ngân sách chỉ cho phép rà một số lượng frame hữu hạn, ưu tiên frame_0369.jpg để tránh dành hai lượt rà cho các frame có khả năng chứa thông tin tương tự. Việc hai ảnh thực sự gần trùng về nội dung cần được xác nhận bằng contact sheet.

Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Không thể chứng minh trực tiếp, vì `round1_diff.md` cho thấy lô 12 ảnh chọn được đã chứa nhiều cố gắng sửa nhãn: model đề xuất 169 box, sau sửa còn 329 box; trong số đó có 56 box giữ nguyên, 87 chỉnh sửa, 26 xoá và 186 thêm mới. Đặc biệt, các ảnh có rank cao như frame_0369.jpg và frame_0182.jpg đều có nhiều thay đổi nhãn: lần lượt từ 14 → 35 box với 23 box được thêm và từ 13 → 25 box với 14 box được thêm. Điều này cho thấy các frame được chọn chứa nhiều trường hợp model và nhãn người gán không thống nhất, đặc biệt ở các object mà model chưa dự đoán đầy đủ. Vì vậy, các frame này phù hợp để ưu tiên rà soát; tuy nhiên, mức độ thay đổi nhãn không tự nó chứng minh rằng việc đưa chúng vào train sẽ cải thiện mô hình.

## 4. Các vòng học chủ động (active learning)

Bảng từ `reports/rounds_table.md`:

| vòng | model                                   | ảnh train | box train |  AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 |    F1 | R small | R medium | R large |
| ---: | --------------------------------------- | --------: | --------: | ----: | -------------------: | -----: | -----: | ----: | ------: | -------: | ------: |
|    0 | yolov8n cold start (COCO car+bus+truck) |         0 |         0 | 0.771 |                    — |  0.925 |  0.489 | 0.640 |   0.182 |    0.547 |   0.561 |
|    1 | yolov8n fine-tune vong 1..1             |        12 |       329 | 0.524 |               -0.247 |  1.000 |  0.037 | 0.072 |   0.000 |    0.030 |   0.146 |

Ở vòng 1, mức độ sửa nhãn rất lớn so với pre-label: theo `outputs/round1_diff.md`, model đề xuất 169 box trên 12 ảnh, sau khi sửa còn 329 box. Chi tiết: 56 box được giữ nguyên, 87 sửa, 26 xóa (FP của model), 186 thêm mới (FN của model). Tỷ lệ chung accept rate là 33%. Điều này cho thấy lô chọn không phải chỉ là dữ liệu “đúng” hơn; nhiều box bị model gợi ý sai hoặc thiếu khiến nhãn người sửa đóng vai trò quyết định. So với AP50, vòng 1 giảm từ 0.771 xuống 0.524, tức giảm 0.247 tuyệt đối, và cũng giảm mạnh so với vòng 0 ở mọi nhóm xe: recall small từ 0.182 xuống 0, medium từ 0.547 xuống 0.030, large từ 0.561 xuống 0.146. Về mặt thực nghiệm, mô hình vòng 1 tốt hơn ở độ chính xác (precision 1.000) nhưng rất tệ ở recall; nghĩa là nó ít cảnh báo sai nhưng lại bỏ hầu hết các xe trong test, đặc biệt là xe nhỏ và xe trung bình.

Mô tả một ca đổi sau fine-tune: trong `compare_round1.jpg`, có một trường hợp xe lớn ở đường giữa ảnh xuất hiện rõ hơn sau train trên một số frame, nhưng đồng thời nhiều xe ở khoảng xa hoặc mép các làn bị model hoàn toàn bỏ mất. Đây là kiểu “tốt hơn một số ca nhưng xấu đi ở nhiều ca còn lại”. Một giả thuyết cần kiểm tra là batch được chọn tập trung vào các frame có nhiều xe tối và xe ở mép ảnh; khi fine-tune, mô hình có thể thích nghi quá mạnh với kiểu cảnh này và giảm khả năng tổng quát hóa sang các cảnh khác trong test. Tuy nhiên, chỉ từ kết quả hiện tại chưa thể xác định đây là nguyên nhân duy nhất của việc recall giảm mạnh.

Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt các loại quan sát:

- Quan sát độc lập trước khi xem pre-label: trong `BLIND_SCAN.md`, tôi giả sử chỉ nhìn bằng mắt đã đếm 26 xe trên `frame_0392.jpg` và nêu rõ các vị trí dễ bị AI bỏ sót ở mép trái giữa ảnh và mép dưới giữa ảnh. Đây là quan sát độc lập trước khi dựa vào model.
- Lỗi pre-label đã sửa: `REVIEW_LOG.csv` cho thấy khớp với trường hợp thực tế: 2 box nhận nhầm xe buýt lớn thành car, 4 box xe con ở mép ảnh cần điều chỉnh, và nhiều box bị dời/bỏ sót đúng theo guideline.
- Kết quả mô hình sau train: `round1_diff.md` và `metrics_round1.json` cho thấy mô hình sau fine-tune không thể phát hiện nhiều xe trong test, dù đã được sửa nhãn trên 12 ảnh. Sự khác biệt giữa quan sát độc lập và kết quả sau train cho thấy mô hình đang học theo kiểu label của batch chứ không tổng quát đúng trên toàn bộ test.

Một ca khó theo guideline là ở `frame_0392.jpg`: xe con ở mép ảnh và xe con gần mép dưới có phần thân bị cắt hoặc tối, nhưng vẫn đủ thông tin để xác định là xe. Guideline quy định không được bỏ sót các xe còn nhìn thấy rõ phần thân, nhưng cũng không được giữ những box quá lệch khỏi thân xe khi bị cắt. Những ca như vậy dễ mắc lỗi pre-label và dễ khiến mô hình trong vòng train bị “học sai” nếu không rà lại kỹ.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start là không cải thiện: AP50 giảm từ 0.771 xuống 0.524, Fall 0.247 absolute; recall giảm từ 0.489 xuống 0.037, và recall theo kích thước đều tụt xuống rất mạnh. Về mặt thực nghiệm, kết quả vòng 1 không cho thấy việc bổ sung lô 12 ảnh đã sửa giúp cải thiện khả năng phát hiện trên bộ đánh giá hiện tại. Vì vậy, với dữ liệu hiện tại, tôi sẽ dừng lại ở vòng 1 và không train thêm ngay lúc này; nếu tiếp tục, điều cần làm trước hết là kiểm tra chất lượng label trước khi train, vì việc sửa nhãn không đồng nghĩa là nhãn đã đúng và đủ đa dạng.

Hai ca còn yếu hoặc bất định cho vòng sau:

1. `frame_0392.jpg`: nhiều xe con ở mép ảnh và xe buýt lớn gần biên; chi phí rà nhãn cao vì cần cẩn thận với box ở mép cắt và xe tối, đồng thời nguy cơ ảnh gần trùng cao vì cùng cảnh nhiều frame liên tiếp có cấu trúc tương tự.
2. `frame_0331.jpg` hoặc `frame_0326.jpg`: đây là các frame có số object lớn và nhiều ambiguous, nên rất phù hợp để sửa thêm nếu muốn tăng đa dạng cảnh và mâu thuẫn giữa model và nhãn. Chi phí rà nhãn vừa phải nhưng rủi ro là các frame gần nhau về thời gian khiến dữ liệu lặp lại, làm giảm hiệu quả học.

Tập kiểm thử chỉ có 20 ảnh; vì vậy các chênh lệch nho nhỏ (dưới khoảng 0.01 AP50) chưa đủ để kết luận mô hình tiến bộ/giảm sút. Ngoài ra, nhãn test do mô hình tạo ra chưa được người rà từng box và 14 box rất nhỏ dưới 16 px được bỏ qua, nên AP50 chỉ là “mức khớp với nhãn tham chiếu do mô hình tạo”, không phải thước đo tuyệt đối về chất lượng thực tế. Ngoài ra, nếu AP50 giảm, tôi sẽ kiểm tra trước khi train thêm các điều sau:

- label trên các frame chọn có đúng guideline và không có box “vớt” quá nhiều/thiếu nhiều không cần thiết;
- dữ liệu có bị lặp hoặc gần trùng trong batch không, dẫn đến overfit cục bộ;
- recall theo kích thước và trường hợp mép ảnh/xe tối có bị sụt đáng kể không;
- xem lại `round1_diff.md` để xác định xem hiệu suất giảm do nhãn sai, do batch không đa dạng, hay do model bị overfit vào một kiểu cảnh.

Vì vậy, vòng 1 cho thấy rằng việc chọn ảnh có độ bất định cao là hợp lý, nhưng lô hiện tại chưa đủ để mang lại cải thiện trên test; cần rà lại nhãn, tăng tính đa dạng và kiểm tra lại giả thiết “mô hình sai” trước khi tiếp tục học chủ động.
