# Phản Tư Về Lakehouse Anti-Pattern (REFLECTION.md)

### Rủi Ro Lớn Nhất: Tích Tụ File Nhỏ & File Rác Chưa Commit (Orphan Files)

Hệ thống LLM Observability của team thu thập dữ liệu inference theo thời gian thực (micro-batch 5 giây ghi raw JSON vào Bronze). Mô hình này đối mặt với hai rủi ro lớn:

1. **Tích tụ File nhỏ**: Hàng nghìn file Parquet dung lượng KB được tạo ra mỗi ngày. Nếu không chạy `OPTIMIZE` định kỳ, chi phí truy vấn tăng vọt do tốn thời gian quét metadata và chi phí request `GET` trên S3.
2. **File rác chưa commit (Orphans)**: Khi worker nạp dữ liệu bị crash giữa chừng, các file Parquet dở dang nằm lại trên đĩa. Do `VACUUM` chỉ xóa các file đã có *tombstone* trong transaction log, các file rác này hoàn toàn "vô hình" với lịch sử nhưng vẫn làm phình hóa đơn lưu trữ S3.

#### Chiến Lược Khắc Phục:
* **Gộp file định kỳ (Compaction & Clustering)**: Chạy `OPTIMIZE` (mục tiêu 128–512 MB) kết hợp `Z-ORDER` theo `user_id` hàng giờ để duy trì tỷ lệ bỏ qua file (pruning) $\ge 10\times$.
* **Dọn dẹp file rác (Orphan Sweeping)**: Thiết lập job quét quét đĩa tự động (với age-guard 24 giờ) kết hợp `expire_snapshots` để thu hồi triệt để dung lượng lưu trữ.
