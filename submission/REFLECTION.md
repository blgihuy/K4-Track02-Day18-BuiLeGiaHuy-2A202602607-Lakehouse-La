# Suy ngẫm về Lakehouse & Khai báo AI (Reflection)

### 1. Lakehouse Anti-pattern: Bất đối xứng vòng đời giữa Lakehouse và Vector DB (Lifecycle Skew)

- **Bối cảnh & Nguyên nhân:** Trong các hệ thống RAG quy mô lớn, kỹ sư thường tách biệt Lakehouse (lưu trữ văn bản) và Vector DB ngoại vi (lưu embeddings) với cơ chế đồng bộ định kỳ một chiều dạng upsert. Khi phát sinh yêu cầu tuân thủ quyền được quên (GDPR Article 17) hoặc thu hồi dữ liệu riêng tư, lệnh `DELETE` được thực thi trên Lakehouse nhưng không được đồng bộ hóa sang Vector DB. Kết quả là Vector DB tiếp tục trả về embeddings của dữ liệu đã xóa, gây rò rỉ dữ liệu nhạy cảm vào context của LLM.
- **Cách phòng tránh:**
  1. **In-table Vectors:** Lưu trữ embeddings trực tiếp trong bảng Delta/Iceberg dưới dạng vector fixed-size array để thao tác xóa mang tính nguyên tử trên toàn bộ bản ghi và vector.
  2. **Change Data Feed (CDF):** Đối với index ngoại vi, bắt buộc pipeline đồng bộ phải tiêu thụ luồng sự kiện `DELETE` từ Delta CDF thay vì chỉ quét upsert định kỳ.

### 2. Khai báo phạm vi sử dụng AI (AI Usage Declaration)
Sử dụng Antigravity IDE (Gemini) để hỗ trợ tự động hóa việc thực thi và lưu trữ output notebook, kiểm tra cú pháp PowerShell trên môi trường Windows và định dạng bảng biểu. Mọi phân tích cơ chế, số liệu đo lường thực tế và quyết định kiến trúc trong bài nộp đều do học viên trực tiếp kiểm chứng và chịu trách nhiệm.
