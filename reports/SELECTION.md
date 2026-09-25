# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

1. `frame_0182.jpg` — điểm 0.9591 — thời điểm 72.8s — rank 1.
   Ưu tiên vì có điểm cao nhất trong 50 dòng đầu và `n_ambiguous=18`, cho thấy có nhiều trường
   hợp cần con người rà lại.

2. `frame_0369.jpg` — điểm 0.9324 — thời điểm 147.6s — rank 2.
   Ưu tiên vì điểm rất cao và có 16 trường hợp ambiguous. Frame `0368` ở 147.2s có rank 9
   nhưng không được chọn; hai frame nằm rất gần nhau về thời gian/thứ tự frame nên cần lưu ý
   khả năng thông tin bị trùng khi phân bổ ngân sách rà.

3. `frame_0380.jpg` — điểm 0.9170 — thời điểm 152.0s — rank 3.
   Ưu tiên vì thuộc nhóm điểm cao nhất và có `n_ambiguous=15`, cung cấp một trường hợp có nhiều
   vị trí cần rà.

4. `frame_0326.jpg` — điểm 0.9155 — thời điểm 130.4s — rank 4.
   Ưu tiên vì điểm cao, có 39 box và 15 trường hợp ambiguous, đại diện cho frame có số lượng
   object tương đối lớn.

5. `frame_0331.jpg` — điểm 0.9154 — thời điểm 132.4s — rank 5.
   Ưu tiên vì nằm trong top 5, có 47 box và `n_ambiguous=18`, là một trong các frame có số
   trường hợp ambiguous cao nhất trong nhóm đầu.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

- `frame_0182.jpg`: rank 1, score 0.9591, thời điểm 72.8s, `n_boxes=28`, `n_ambiguous=18`,
  `selected=True`.
- `frame_0369.jpg`: rank 2, score 0.9324, thời điểm 147.6s, `n_boxes=43`,
  `n_ambiguous=16`, `selected=True`.
- `frame_0380.jpg`: rank 3, score 0.9170, thời điểm 152.0s, `n_boxes=40`,
  `n_ambiguous=15`, `selected=True`.

CSV cho thấy tổng cộng 12 frame được đánh dấu `selected=True` trong 50 dòng đầu; ba frame
trên là ba frame có rank cao nhất trong lô được chọn.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

`frame_0372.jpg` — điểm 0.9101 — thời điểm 148.8s — rank 6 — `selected=False`.
Đây là một frame có điểm cao nhưng không được chọn. Nó nằm rất gần `frame_0369.jpg`
(147.6s) về thời điểm, trong khi `frame_0369.jpg` đã được chọn. Vì ngân sách chỉ cho phép
rà một số lượng frame hữu hạn, đây là ứng viên có thể bị loại để tránh dành ngân sách cho các
frame gần nhau. Tuy nhiên, CSV chỉ chứng minh sự gần nhau về thời gian/frame; cần xem
contact sheet để kết luận hai ảnh thực sự gần trùng về nội dung.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

Phép chọn này chỉ là cơ chế ưu tiên frame cần rà soát dựa trên score, số box và số trường hợp
ambiguous trong `selection_round1.csv`. Nó không phải phép đánh giá độc lập chất lượng mô hình.
Đặc biệt, không thể từ việc một frame được chọn hay không được chọn kết luận rằng model đúng
hay sai trên frame đó. Việc xác nhận chất lượng cần dựa trên kết quả đối chiếu nhãn sau khi rà
và các metric/đánh giá phù hợp.
