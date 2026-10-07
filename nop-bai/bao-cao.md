# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Mạnh Hải |
| MSSV | 2A202602988 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/NguyenManhHai2004/K4-L3-DAY21-NguyenManhHai-2A202602988-CI-CD-for-AI-Systems |
| Ngày nộp | 08/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Lần 3 có f1_score cao nhất (0,7149) và vượt ngưỡng 0,65 của Quality Gate. Lần có accuracy cao nhất lại là lần 1 (0,8780) với f1_score thấp hơn, nên accuracy không phản ánh đúng khả năng bắt lớp thu nhập cao. Lần 2 dùng ít cây, cây nông và learning_rate nhỏ nên f1_score chỉ đạt 0,6051, dưới ngưỡng. Vì mỗi lần tôi đổi cả ba tham số, tôi chưa tách riêng được tác động của n_estimators và learning_rate.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Trong tập Adult, lớp thu nhập cao chỉ chiếm khoảng 24,8%, lớp thu nhập thấp chiếm 75,2%. Một mô hình luôn đoán "thu nhập thấp" vẫn đạt accuracy 75,2% dù vô dụng. F1 của lớp dương là trung bình điều hòa giữa precision và recall, nên đo đúng khả năng bắt trúng mà không bỏ sót lớp thu nhập cao. Nếu dùng average "weighted" hoặc "macro", lớp đa số sẽ kéo điểm lên.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| DVC push và upload mô hình báo lỗi chunked encoding. | boto3 mới bật checksum dạng chunked mà S3 API của Oracle chưa hỗ trợ. | Đặt hai biến AWS checksum thành when_required. |
| Không gọi được cổng 8080 dù đã mở Security List. | iptables của Ubuntu trên OCI có quy tắc REJECT đứng trước quy tắc mở cổng. | Đưa quy tắc ACCEPT cổng 8080 lên trước và lưu bằng netfilter-persistent. |
| Job Train lỗi InvalidBucketName, job Release lỗi xác thực SSH. | Secret ARTIFACT_BUCKET dính ký tự xuống dòng và SERVER_SSH_KEY dán sai nội dung. | Nhập lại hai secret rồi chạy lại workflow. |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0,7149 | 0,8740 |
| Bước 3 (thêm `train_batch2`) | 0,7354 | 0,8820 |

**Nhận xét:** Khi tăng dữ liệu từ 22.361 lên 44.722 mẫu, f1_score tăng khoảng 0,02 và accuracy tăng 0,008 trên cùng tập holdout. Mức tăng nhỏ vì hai nửa dữ liệu cùng phân phối, và holdout chỉ có 500 mẫu nên có thể là dao động ngẫu nhiên. Điều được kiểm chứng chắc chắn là quy trình: một commit file `.dvc` đã tự kích hoạt cả bốn job và triển khai mô hình mới lên VM.
