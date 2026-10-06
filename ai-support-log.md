# AI Support Log

## Thông tin

- **Học viên:** Nguyễn Thị Bảo Trang
- **MHV:** 2A202602580
- **Bài:** Track 1 · Day 20 · Product Metrics
- **Ngày thực hiện:** 06/10/2026
- **Công cụ AI:** OpenAI Codex trong ứng dụng Codex

## AI Support Log

**AI đã giúp tôi ở đâu?**

AI giúp tôi brainstorm các ứng viên core action cho ViVi, gồm hành vi xác nhận
nhu cầu, xem xét đề xuất xe, đặt lịch lái thử và tư vấn viên hoàn thành review
cho lead. AI đặt các câu hỏi phản biện để tôi kiểm tra hành vi nào thực sự tạo
value, hành vi nào chỉ là thao tác giao diện hoặc output của hệ thống.

AI cũng gợi ý một số tên event dạng `object_action`, như
`needs_confirmed`, `recommendation_reviewed`, `test_drive_requested` và
`lead_review_completed`, cùng acceptance criteria về thời điểm hoàn tất và
chống ghi trùng do reload hoặc retry.

Cuối cùng, AI đóng vai khách hàng khó tính để chất vấn liệu core job có phản ánh
đúng vấn đề của người dùng và cadence có xuất phát từ nhu cầu tự nhiên hay chỉ
được chọn cho thuận tiện khi làm dashboard.

**AI sai, hời hợt hoặc đề xuất metric sai nature ở đâu?**

AI ban đầu ưu tiên hành vi của tư vấn viên vì hành vi này dễ lặp lại và dễ xây
retention hơn. Cách tiếp cận đó có nguy cơ chọn hành vi vì dễ đo thay vì vì gần
core value nhất. AI cũng đề xuất cadence theo tuần và các ngưỡng như 3
lead/tuần, SLA 1 ngày hoặc quality window 7 ngày khi chưa có dữ liệu thực tế để
chứng minh. Vì vậy tôi không sử dụng các con số này như benchmark hay kết luận
chính thức.

**Tôi đã tự sửa hoặc quyết định lại điều gì?**

Tôi chọn coe action là Advisor hoàn thành human review cho đề xuất của một lead đủ điều kiện và chốt next step.

- Actor: Tư vấn viên VinFast.
- Object: Lead đã có consent và recommendation hiện hành.
- Completion rule: Advisor approve/edit/reject đề xuất, ghi lý do khi cần, chọn next step và hệ thống lưu thành công review cùng audit log.
- Candidate event: lead_review_completed.
Lý do lúc đó tôi chọn phương án này:
- Gắn với core job của phía Sales: hiểu đúng khách và tiếp tục chăm sóc mà không bắt khách kể lại.
- Lặp tự nhiên theo mỗi lead nên cadence và retention không bị ép.
- Quan sát được thời điểm hoàn tất.
- Không phải thao tác “mở lead” hoặc output “AI tạo đề xuất”.
- Team có thể cải thiện context, review queue và workflow.

Tôi tự kết luận natural cadence là Đối với tư vấn viên VinFast được giao lead online, core action hoàn thành human review và chốt next step cho một lead đủ điều kiện thường xuất hiện nhiều lần trong các ngày làm việc có lead mới hoặc lead thay đổi, vì mỗi assignment hoặc recommendation version mới tạo ra một nghĩa vụ xử lý độc lập. Do đó, nhịp đo phù hợp là theo tuần làm việc ở cấp Advisor account.

Phân biệt ngắn gọn:
- Natural trigger: Điều gì khiến hành vi bắt đầu ngay lúc này?
- Repeat condition: Khi nào hành vi có lý do xuất hiện thêm lần nữa?
- Cadence: Theo nhịp nào nên tổng hợp và đánh giá hành vi?
Notification không phải natural trigger. Nó chỉ nhắc Advisor về lead; lý do thật sự để quay lại là có lead mới hoặc dữ liệu của lead đã thay đổi.

Tôi tự viết metric hypothesis:

Nếu shared customer memory, provenance và review queue của loop hoạt động, Weekly Qualified Lead Journeys Progressed after Human Review sẽ tăng so với baseline trong 4 tuần triển khai, vì Advisor có đủ context để hoàn thành review và chốt next step nhanh hơn; đồng thời customer correction rate không được tăng.

Các thành phần:
- Loop được kiểm chứng: shared customer memory + provenance + review queue.
- Metric chính: số lead đủ điều kiện mỗi tuần được tiến triển sau human review.
- Hướng thay đổi: tăng so với baseline.
- Khung thời gian: 4 tuần.
- Cơ chế: Advisor có đủ bối cảnh nên review và chốt bước tiếp theo nhanh hơn.
- Counter-metric: tỷ lệ khách phải sửa lại thông tin không được tăng.

Tôi loại bỏ các benchmark và ngưỡng chưa có nguồn. Những giá trị cần xác định
sau được ghi là giả thuyết cần kiểm chứng bằng phỏng vấn hoặc dữ liệu tracking
thực tế.
