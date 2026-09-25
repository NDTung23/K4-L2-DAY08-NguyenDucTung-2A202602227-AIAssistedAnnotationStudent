# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Đức Tùng

Công cụ gán nhãn đã dùng: CVAT (chạy bằng Docker trên máy cá nhân)

## 1. Dữ liệu và cách chia tập

Camera trong video đứng yên và quay liên tục, nên hai khung hình cách nhau chỉ 0.4 giây gần như giống hệt nhau, và mỗi chiếc xe xuất hiện liên tục trong nhiều giây liền. Nếu chia ngẫu nhiên, rất có thể cùng một chiếc xe sẽ xuất hiện ở cả tập huấn luyện lẫn tập kiểm thử — mô hình khi đó được chấm điểm trên chính chiếc xe nó đã "nhìn thấy" lúc học, khiến số đo (AP50, recall...) bị đẩy cao hơn thực tế. Đây là hiện tượng rò rỉ dữ liệu (data leakage). Việc chia theo trục thời gian, kèm vùng đệm 4 giây quanh mỗi đoạn test, đảm bảo ảnh pool gần ảnh test nhất vẫn cách xa 4.4 giây — đủ để tránh hiện tượng này.

## 2. Mô hình khởi đầu lạnh (cold start)

Từ `rounds_table.md`:
> vòng 0 | yolov8n cold start (COCO car+bus+truck) | 0 ảnh train | 0 box train | AP50 = 0.771 | P@0.25 = 0.925 | R@0.25 = 0.489 | R small = 0.182 | R medium = 0.547 | R large = 0.561

Nhìn `outputs/compare_round0.jpg` (và panel giữa của `compare_round1.jpg`), mô hình cold start bỏ sót (FN, box vàng) chủ yếu là các xe ở làn xa, xe chỉ còn thấy đèn hậu/đèn pha lóa sáng và xe bị xe khác che một phần — ví dụ ở frame_0350, cold start bỏ sót tới 14/23 xe, tập trung ở cụm xe xa phía trên ảnh. Recall theo kích thước xe cho thấy rõ điều này: recall xe nhỏ chỉ 0.182, trong khi xe vừa và lớn lần lượt là 0.547 và 0.561 — mô hình gần như "mù" với xe nhỏ/xa, đúng như quan sát bằng mắt.

Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai: ở một số ảnh so sánh (ví dụ frame_0250), có 2 box đỏ (false positive) nằm ở khu vực nhiều xe đứng sát/chồng đèn hậu lên nhau. Rất có thể đây không phải là model "bịa" ra xe, mà là trường hợp nhãn tham chiếu gộp 2 xe sát nhau thành 1 box (đúng loại lỗi mà `GUIDELINE_LABEL.md` cảnh báo), khiến box thứ hai của model bị tính nhầm là false positive. Cần xem trực tiếp ảnh gốc khu vực đó trước khi kết luận model dự đoán sai.

## 3. Chiến lược chọn mẫu

Công thức `score = W_U·U + W_A·A + W_D·D` (mặc định W_U=0.5, W_A=0.3, W_D=0.2) kết hợp ba tín hiệu: **U** — mức độ không chắc chắn trung bình của 5 box có độ tin cậy thấp nhất trong ảnh; **A** — tỉ lệ box nằm trong vùng "mơ hồ" (độ tin cậy 0.15–0.5) so với ảnh có nhiều box mơ hồ nhất; **D** — khoảng cách thời gian tới ảnh gần nhất đã được gán nhãn (ở vòng đầu, D=1 cho mọi ảnh vì chưa có ảnh nào được gán). `MIN_GAP_S` giới hạn khoảng cách thời gian tối thiểu giữa các ảnh được chọn, để tránh chọn 2 ảnh gần như giống hệt nhau trong cùng một lô.

Ba ví dụ từ `SELECTION.md`: `frame_0182.jpg` (hạng 1) có cả U và A đều rất cao nên đứng đầu; `frame_0099.jpg` (hạng 8) có U cao nhất toàn danh sách (0.946) nhưng A thấp hơn nên bị kéo xuống hạng 8, cho thấy U một mình không quyết định thứ hạng; `frame_0392.jpg` (hạng 15, xếp cuối trong 12 ảnh được chọn) có U cao nhất trong top 15 (0.9747) nhưng A thấp nhất (0.6667). Một ví dụ khác: `frame_0372.jpg` (hạng 6, điểm 0.9101) bị loại dù điểm gần ngang `frame_0331.jpg` (hạng 5), vì thời điểm của nó (148.8s) chỉ cách ảnh đã chọn `frame_0369.jpg` (147.6s) đúng 1.2 giây — minh hoạ vai trò của `MIN_GAP_S` trong việc tránh trùng lặp.

Điểm bất định (U, A) không chứng minh ảnh đó sẽ cải thiện mô hình — nó chỉ cho biết mô hình "phân vân" ở ảnh đó, có thể vì ảnh nhiều xe khó, cũng có thể chỉ vì ảnh nhoè/chất lượng kém. Việc sửa nhãn đúng ở ảnh điểm cao không đảm bảo mô hình học tốt hơn sau fine-tune, như kết quả vòng 1 dưới đây cho thấy.

## 4. Các vòng học chủ động (active learning)

| vòng | model | ảnh train | box train | AP50 | Δ AP50 | P@0.25 | R@0.25 | R small | R medium | R large |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 360 | 0.329 | **-0.442** | 1.000 | 0.030 | 0.000 | 0.020 | 0.146 |

Theo `outputs/round1_diff.md` (và log console khi chạy `pack_labels.py`): trong 169 box AI đề xuất ban đầu, tôi giữ nguyên (accepted) 150, chỉnh sửa (edited) 10, xoá (deleted) 9, và thêm mới (added) 200 box còn thiếu — tổng cộng 360 box trên 12 ảnh.

AP50 sau fine-tune **giảm mạnh** từ 0.771 xuống 0.329 (Δ = -0.442). Recall gần như sụp đổ ở mọi nhóm kích thước xe: nhỏ từ 0.182 → 0.000, vừa từ 0.547 → 0.020, lớn từ 0.561 → 0.146. Trong khi đó Precision lại tăng lên 1.000 — mô hình gần như "không dám" đoán, nhưng những gì nó đoán đều đúng. Đây là dấu hiệu điển hình của hiện tượng **catastrophic forgetting** (quên thảm khốc): huấn luyện với 50 epoch trên vỏn vẹn 12 ảnh khiến mô hình học lệch quá sâu vào tập nhỏ này và quên phần lớn kiến thức tổng quát đã có từ trước.

Một ca cụ thể từ `outputs/compare_round1.jpg`: ở frame_0050, cold start đạt TP=11, FP=2, FN=7 (khá tốt trong 19 xe tham chiếu), nhưng sau fine-tune vòng 1 chỉ còn TP=1, FP=0, FN=17 — kết quả xấu đi rõ rệt và có thể kiểm chứng trực tiếp bằng mắt trên ảnh so sánh.

Phân biệt ba loại quan sát: `BLIND_SCAN.md` ghi lại quan sát độc lập của tôi trên `frame_0099.jpg` (đếm ~25 xe bằng mắt, trước khi xem nhãn AI) — đây là bước kiểm tra độc lập, không liên quan đến việc mô hình đúng hay sai. `REVIEW_LOG.csv` ghi lại các lỗi pre-label cụ thể đã sửa, ví dụ ở `frame_0107.jpg` và `frame_0182.jpg`, AI gộp nhiều xe đứng sát nhau vào chung 1 box — đây là lỗi ở bước gán nhãn, đã được sửa đúng theo `GUIDELINE_LABEL.md`. Còn kết quả AP50 giảm ở trên là lỗi phát sinh **sau** khi nhãn đã sửa đúng, ở bước huấn luyện — không liên quan đến chất lượng nhãn.

Một ca khó theo guideline: ở `frame_0099.jpg`, có xe bị cây che gần hết, chỉ còn thấy một đèn pha và khoảng 25% thân xe. Theo `GUIDELINE_LABEL.md` (mục "xe bị xe khác che một phần" áp dụng tương tự cho vật cản khác), tôi vẫn gán nhãn cho phần thân xe nhìn thấy được, vì vẫn đoán được đường viền quanh đèn.

## 5. Kết luận và giới hạn

Kết quả vòng 1 **kém hơn hẳn** cold start, không phải do nhãn sai mà do quá trình fine-tune (50 epoch trên 12 ảnh) làm mô hình quên kiến thức cũ. Mức giảm AP50 (-0.442) lớn hơn nhiều so với ngưỡng nhiễu 0.01 mà `data/DATA.md` cảnh báo với tập test chỉ 20 ảnh, nên đây là tín hiệu thật, không phải nhiễu ngẫu nhiên.

**Tôi đề xuất dừng lại ở đây để kiểm tra cấu hình huấn luyện** (số epoch, learning rate, có nên đóng băng một phần backbone) trước khi làm vòng 2, thay vì tiếp tục gán nhãn thêm trên một mô hình đã bị lỗi huấn luyện — vì làm vòng 2 ngay bây giờ sẽ tiếp tục fine-tune trên một checkpoint đã hỏng, không phản ánh đúng hiệu quả của việc chọn mẫu chủ động.

Hai ca đề xuất cho vòng sau (khi cấu hình huấn luyện đã được sửa):
- `frame_0392.jpg` — điểm U cao nhất trong 12 ảnh đã chọn nhưng A thấp nhất, đáng xem lại xem việc sửa nhãn ở đây có thực sự đóng góp nhiều thông tin mới không. Chi phí rà thấp (đã có sẵn trong lô, không cần chọn thêm).
- `frame_0372.jpg` hoặc `frame_0368.jpg` — điểm cao nhưng bị loại vì gần trùng thời gian với ảnh đã chọn; nên cân nhắc xem nội dung có thực sự trùng lặp hay có góc nhìn khác biệt đáng gán nhãn, tránh lãng phí công nếu chọn nhầm ảnh gần như giống hệt ảnh đã có.

Giới hạn của kết luận: tập test chỉ có 20 ảnh (417 box, bỏ qua 14 box quá nhỏ), nên các so sánh chi tiết theo từng nhóm xe có thể không ổn định. Nhãn tham chiếu do một mô hình khác tạo ra và **chưa được người rà từng box** — nên AP50 chỉ đo mức khớp với bộ tham chiếu này, không phải thước đo tuyệt đối về độ chính xác thực tế; một số box đỏ (FP) có thể thực chất là nhãn tham chiếu bị thiếu/gộp sai (như nêu ở mục 2).

Nếu AP50 giảm, trước khi train thêm tôi sẽ kiểm tra theo thứ tự: (1) định dạng và tọa độ nhãn mới đóng gói có đúng chuẩn YOLO không (đã xác nhận đúng ở vòng này); (2) siêu tham số huấn luyện — đặc biệt số epoch và learning rate — có phù hợp với lượng ảnh train nhỏ không; (3) so sánh trực quan qua `compare_round*.jpg` xem lỗi là do mô hình bỏ sót toàn bộ (như trường hợp này) hay chỉ lệch nhẹ ở một vài nhóm xe cụ thể.