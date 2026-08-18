# Phản Tư Về Lakehouse Anti-Pattern

*Nền tảng bảng: Delta Lake*

### Rủi Ro Lớn Nhất

Micro-batch 5 giây ghi raw JSON vào Bronze tạo ra hai rủi ro:

1. **File nhỏ tích tụ**: hàng nghìn file Parquet cỡ KB/ngày. Không `OPTIMIZE` định kỳ → chi phí quét metadata và request `GET` trên S3 tăng vọt.
2. **File rác chưa commit**: worker crash giữa chừng để lại file Parquet dở dang, chưa từng vào transaction log. `VACUUM` xóa được cả file này lẫn file tombstone cũ, nhưng nếu không chạy định kỳ, chúng "vô hình" với lịch sử truy vấn mà vẫn tốn phí lưu trữ.

### Chiến Lược Khắc Phục

* **`OPTIMIZE`** (128–512 MB/file) + **`ZORDER BY (user_id)`** hàng giờ để tăng tỷ lệ bỏ qua file khi truy vấn.
* **`VACUUM`** định kỳ với retention dài hơn job đọc đang chạy lâu nhất, tránh xóa nhầm file đang được tham chiếu.

*(Nếu dùng Iceberg: thay bằng `rewrite_data_files`, `remove_orphan_files`, `expire_snapshots` — ba lệnh riêng biệt.)*