# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét ảnh gần trùng hoặc trường hợp model không dự đoán được box:

1. **frame_0182.jpg** (hạng 1, điểm 0.9591, t=72.8s) — Điểm cao nhất trong 50 ảnh, U=0.9182 rất cao, 28 box với 18 box mơ hồ (A=1.0).

2. **frame_0369.jpg** (hạng 2, điểm 0.9324, t=147.6s) — Điểm cao thứ 2. Đây là quyết định xét ảnh gần trùng: `frame_0368.jpg` (hạng 9, điểm 0.9003, t=147.2s) chỉ cách 0.4 giây, gần như cùng khung cảnh. Ưu tiên `frame_0369` vì điểm cao hơn, tránh tốn ngân sách rà 2 ảnh gần trùng.

3. **frame_0380.jpg** (hạng 3, điểm 0.917, t=152.0s) — Điểm cao, 40 box, A=0.8333.

4. **frame_0326.jpg** (hạng 4, điểm 0.9155, t=130.4s) — Điểm cao, U và A đều trên 0.83.

5. **frame_0312.jpg** (hạng 7, điểm 0.91, t=124.8s) — A=1.0 (100% box có độ tin cậy 0.15–0.5), nhiều box mơ hồ nhất trong top này.

Trong top 50, không có ảnh nào `empty=True` (model không đoán được box nào), nên lý do "ảnh gần trùng" ở mục 2 được dùng để đáp ứng yêu cầu này.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **frame_0182.jpg** (hạng 1) — Điểm cao nhất (0.9591); U=0.9182, A=1.0 (toàn bộ 28 box đều có độ tin cậy 0.15–0.5); 18/28 box mơ hồ. Đây là ảnh model "phân vân" nhiều nhất cả về độ chắc chắn lẫn tỉ lệ box mơ hồ.

2. **frame_0099.jpg** (hạng 8) — U=0.946, cao nhất trong toàn bộ danh sách, nhưng A chỉ 0.7778 (thấp hơn top 5) nên tổng điểm bị kéo xuống hạng 8. Minh hoạ cách công thức trọng số hoạt động: độ bất định cực cao vẫn có thể xếp hạng thấp hơn nếu tỉ lệ box mơ hồ ít hơn.

3. **frame_0392.jpg** (hạng 15) — U=0.9747, cao nhất trong top 15, nhưng A chỉ 0.6667 (thấp nhất trong top 15) nên bị đẩy xuống hạng cuối trong 12 ảnh được chọn. Cho thấy A và U không phải lúc nào cũng đồng biến.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

**frame_0372.jpg** (hạng 6, điểm 0.9101, U=0.9202) có điểm gần ngang `frame_0331.jpg` (hạng 5, điểm 0.9154) nhưng không được chọn. Lý do nhiều khả năng: thời điểm của nó (148.8s) rất gần `frame_0369.jpg` đã được chọn ở hạng 2 (147.6s, cách 1.2 giây) — công cụ dùng `MIN_GAP_S` để tránh chọn 2 ảnh quá gần nhau về thời gian, nên ưu tiên `frame_0369` (điểm cao hơn) và bỏ qua `frame_0372`.

Điều phép chọn này chưa chứng minh được về chất lượng mô hình:

Điểm số (U, A, D) chỉ phản ánh việc model "không chắc chắn" hoặc có nhiều box ở vùng nhập nhằng — không đảm bảo ảnh đó thực sự hữu ích để cải thiện model, cũng không chứng minh nhãn sau khi sửa sẽ chính xác hơn hay model sẽ học tốt hơn. Một ảnh điểm cao có thể chỉ là ảnh nhoè, nhiều xe bị che khuất khó gán nhãn đúng — chọn ảnh này chỉ là bước "nên xem", không phải "sẽ giúp ích".