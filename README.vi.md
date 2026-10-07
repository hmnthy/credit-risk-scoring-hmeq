[Français](README.md) · [English](README.en.md) · **Tiếng Việt**

# Chấm điểm rủi ro tín dụng – HMEQ

Đồ án nhóm thuộc chương trình Master 2 MOSEF (Mô hình hoá thống kê trong kinh tế và tài chính), năm học 2024-2025.

Đồ án dự đoán khả năng vỡ nợ của 5 960 khoản vay thế chấp bằng bất động sản. Từ kết quả đó, nhóm xây dựng một bảng phân khúc rủi ro để hỗ trợ chính sách cấp tín dụng.

## Kết quả chính

Tất cả mô hình được đánh giá trên cùng một tập test (1 192 khoản vay, trong đó 265 khoản vỡ nợ), với ngưỡng quyết định 0,5:

| Mô hình | Recall | Precision | F1 | ROC-AUC |
|---|---|---|---|---|
| Hồi quy logistic (Lasso, biến rời rạc hoá, class weights) | 0,69 | 0,43 | 0,53 | 0,79 |
| **XGBoost + class weights** | **0,80** | **0,89** | **0,84** | **0,96** |
| LightGBM + class weights | 0,81 | 0,85 | 0,83 | 0,96 |

- Trên tập test, XGBoost phát hiện 212/265 khoản vỡ nợ và báo nhầm 27 trường hợp trong số 927 khách hàng trả nợ tốt.
- Khi hạ ngưỡng về điểm Youden trên đường ROC (≈ 0,27), recall tăng lên 0,89 và precision còn 0,77. Đặt ngưỡng ở đâu vẫn là quyết định nghiệp vụ.

## Bài toán

Recall là chỉ số ưu tiên, vì bỏ sót một khoản vỡ nợ tốn kém hơn nhiều so với từ chối nhầm một khách hàng tốt. Dữ liệu mất cân bằng (khoảng 20 % vỡ nợ) và có nhiều giá trị thiếu, nhất là ở tỷ lệ nợ trên thu nhập `DEBTINC` (thiếu 21 %).

## Dữ liệu

Nhóm dùng bộ dữ liệu công khai **HMEQ** (Home Equity): 5 960 khoản vay, 12 biến giải thích và biến mục tiêu nhị phân `BAD` (1 = vỡ nợ, 1 189 trường hợp).

| Biến | Mô tả |
|---|---|
| `LOAN` | Số tiền vay |
| `MORTDUE` | Dư nợ thế chấp hiện tại |
| `VALUE` | Giá trị bất động sản |
| `REASON` | Mục đích vay: `DebtCon` (gộp nợ) hoặc `HomeImp` (sửa chữa nhà) |
| `JOB` | Nhóm nghề nghiệp (6 nhóm) |
| `YOJ` | Số năm làm công việc hiện tại |
| `DEROG` | Số lần ghi nhận vi phạm tín dụng nghiêm trọng |
| `DELINQ` | Số hạn mức tín dụng bị trễ hạn |
| `CLAGE` | Tuổi của hạn mức tín dụng lâu nhất (tháng) |
| `NINQ` | Số lần xin tín dụng gần đây |
| `CLNO` | Số hạn mức tín dụng |
| `DEBTINC` | Tỷ lệ nợ trên thu nhập |

## Cách tiếp cận

1. **Phân tích khám phá:** nhóm xem phân phối đơn biến và hai biến, tỷ lệ vỡ nợ theo từng nhóm và từng khoảng giá trị, và mức chồng lấn giữa hai lớp (pairplot). Báo cáo profiling đầy đủ nằm trong `hmeq_report.html`.
2. **Sàng lọc mô hình:** dùng LazyPredict để chạy khoảng 30 bộ phân loại với 4 cách tiền xử lý: mặc định, loại outlier (IQR), tiền xử lý tối ưu (impute lặp, chuẩn hoá, one-hot), và tiền xử lý tối ưu kết hợp SMOTE. Các mô hình được so sánh theo recall, ROC-AUC và thời gian chạy. Biến `MORTDUE` bị loại vì tương quan mạnh với `VALUE`.
3. **Hồi quy logistic (mô hình tham chiếu):** nhóm kiểm tra các giả định (tuyến tính của logit, VIF, Cramér's V) và thử ba bộ biến: biến gốc, biến rời rạc hoá, biến kết hợp. Mô hình dùng regularization Ridge, Lasso và Elastic Net; mất cân bằng lớp được xử lý bằng class weights hoặc SMOTE.
4. **Gradient boosting (XGBoost, LightGBM):** GridSearchCV (cross-validation phân tầng 5 fold, tối ưu theo recall) tìm cùng lúc siêu tham số và cách tiền xử lý. Nhóm so sánh ba phương án: không cân bằng lớp, SMOTE, và class weights (`scale_pos_weight`). Phương án class weights được giữ lại vì cho recall tốt nhất mà không cần sinh dữ liệu tổng hợp.
5. **Bảng điểm (grille de score):** tỷ lệ vỡ nợ quan sát trên tập train theo nghề nghiệp × mục đích vay × số sự cố (`JOB` × `REASON` × `DEROG` × `DELINQ`, 203 phân khúc), kèm tỷ trọng của từng phân khúc.

## Bảng điểm cho thấy gì

- Ít rủi ro nhất là nhân viên văn phòng (`Office`) chưa có sự cố: 7,8 % vỡ nợ khi vay gộp nợ, 1,2 % khi vay sửa nhà.
- Rủi ro cao hơn là nhân viên kinh doanh (`Sales`, 23 % khi vay gộp nợ) và người tự kinh doanh (`Self`, 18 %).
- Sự cố nghiêm trọng đầu tiên (`DEROG` = 1) làm rủi ro tăng rõ rệt. Từ `DEROG` ≥ 2 hoặc `DELINQ` > 2, tỷ lệ quan sát thường lên tới 100 %, nhưng mỗi phân khúc này chiếm dưới 0,1 % số khoản vay.

## Giới hạn

- Bảng điểm dựa trên **tỷ lệ vỡ nợ quan sát được**, không dựa trên điểm số của mô hình. Các phân khúc quá nhỏ (tỷ lệ 0 % hoặc 100 %) không đáng tin cậy, cần gộp hoặc làm trơn trước khi dùng thực tế.
- Kết quả chỉ dựa trên một lần chia train/test 80/20 (không phân tầng) và không có kiểm định ngoài kỳ, vì HMEQ không có chiều thời gian.
- Kết quả được báo cáo ở ngưỡng 0,5. Nhóm chưa xác định ngưỡng theo ma trận chi phí thực tế.
- Hồi quy logistic và các mô hình boosting dùng cách tiền xử lý khác nhau. Vì vậy, bảng so sánh đặt cấu hình tốt nhất của mỗi họ mô hình cạnh nhau.
- Notebook được viết cho Google Colab (đọc dữ liệu từ Google Drive) và chưa cố định phiên bản môi trường.

## Nhóm thực hiện

Nhóm gồm Sara LAVAL-JEANTET, Chahla TARMOUN, Minh Nhat Thy HUYNH và Salma BENMOUSSA, dưới sự hướng dẫn của thầy Ibrahim TOURE.

Phần việc của Minh Nhat Thy HUYNH: sàng lọc mô hình (LazyPredict, so sánh các cách tiền xử lý), xây dựng mô hình XGBoost và LightGBM, và lập các bảng điểm.

## Nội dung repo

| File | Nội dung |
|---|---|
| `Scoring.ipynb` | Notebook đầy đủ: khám phá dữ liệu, mô hình hoá, đánh giá, bảng điểm |
| `Rapport_Scoring.docx` | Báo cáo đồ án (tiếng Pháp) |
| `hmeq_report.html` | Báo cáo profiling dữ liệu (ydata-profiling); tải về rồi mở bằng trình duyệt |
| `grille_de_score_weight.xlsx` | Bảng điểm đã xuất (203 phân khúc) |
| `requirements.txt` | Thư viện và phiên bản của môi trường gốc |

## Chạy notebook

1. Tải bộ dữ liệu công khai HMEQ (`hmeq.csv`), ví dụ trên [Kaggle](https://www.kaggle.com/datasets/ajay1735/hmeq-data).
2. Mở `Scoring.ipynb` trong Google Colab.
3. Trong ô « Chargement des données » (tải dữ liệu), sửa lại đường dẫn file, hoặc dùng tuỳ chọn tải file thủ công có sẵn trong ô đó.

Môi trường gốc: Google Colab, Python 3.10. Phiên bản các thư viện được liệt kê trong `requirements.txt`.
