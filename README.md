# Track 1 · Day 20 · Product Metrics

- **Họ tên:** Nguyễn Thị Bảo Trang
- **MHV:** 2A202602580
- **Dự án:** ViVi — Trợ lý tư vấn và vận hành bán hàng VinFast
- **Use case được phân tích:** Tư vấn viên tiếp nhận lead đã có consent, kiểm tra đề xuất của AI và đưa lead sang bước tiếp theo phù hợp.
- **Metrics Pack:** [Xem bài làm hoàn chỉnh](METRICS_PACK.md)
- **Tài liệu dự án:** [ViVi Product README](Project/README.md) · [Architecture](Project/ARCHITECTURE.md) · [Prototype](https://linkprototype.vercel.app)
- **AI Support Log:** [Xem nhật ký sử dụng AI](ai-support-log.md)

## Điều tôi mang về để áp dụng cho dự án thật

Không đo ViVi bằng số lượt chat hay thời gian ở trong ứng dụng. Metric phải bắt
đầu từ một hành vi tạo giá trị có thể kiểm chứng: tư vấn viên dùng đúng bối cảnh
khách đã xác nhận, hoàn thành human review và đưa lead sang một next step hợp lệ.
Retention cũng phải điều kiện hóa theo số lead đủ điều kiện được giao; nếu không,
dashboard sẽ phạt tư vấn viên chỉ vì tuần đó không có đủ lead.

## Trạng thái dữ liệu

ViVi hiện là frontend prototype; backend, database và event pipeline chưa được
tích hợp. Vì vậy các định nghĩa metric và tracking contract trong bài là yêu cầu
để triển khai. Baseline và target được để ở trạng thái `CẦN ĐO`, không suy đoán
từ seed data của prototype.
