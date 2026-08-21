# Reflection — Lab 21

**1. Điều gì làm bạn ngạc nhiên nhất?**

Prompt tối ưu đưa target từ 0 lên 0.6875 và format từ 0 lên 1 mà không cập nhật trọng
số. `attn_only` có train loss thấp hơn cấu hình đúng nhưng chỉ hoà target; loss huấn
luyện rõ ràng không phải bảng xếp hạng chất lượng.

**2. Bạn mất nhiều thời gian nhất ở đâu? Nó có phải chỗ bạn dự đoán không?**

NB4 lâu nhất: ba run đối chứng mất khoảng 47.5 phút, pipeline mất 72.4 phút trên T4.
Tôi nghĩ tải model hoặc NB3 sẽ chiếm nhiều nhất, nhưng ba lần nạp và train độc lập mới
là chi phí chính.

**3. Trước lab này bạn tin điều gì về fine-tuning mà giờ bạn không còn tin?**

Tôi từng nghĩ rank lớn hoặc train loss thấp gần như đồng nghĩa model tốt hơn. Kết quả
cho thấy placement, learning rate, mask và metric tác vụ quan trọng hơn; rank chỉ tăng
năng lực biểu diễn chứ không tạo thêm thông tin từ dữ liệu.

**4. Bạn dùng AI assistant vào việc gì trong lab? Chỗ nào nó sai?**

Tôi dùng AI assistant để đọc dự án, đối chiếu artefact, chạy test, tìm lỗi gatekeeper
và soạn báo cáo từ số đo. Ban đầu assistant xem “3 test fail” trong output Colab cũ là
trạng thái hiện tại; chạy lại workspace cho thấy 118/118 test đều xanh. Tôi học được
rằng log cũ và bản tóm tắt phải được kiểm chứng trực tiếp.

**5. Nếu ngày mai phải fine-tune cho một khách hàng thật, bước đầu tiên bạn làm là gì?**

Tôi sẽ định nghĩa tập eval đóng băng và baseline prompt mạnh trước khi train. Sau đó
kiểm tra chat template và giải mã loss mask trên một batch thật, vì nếu chúng sai thì
thời gian GPU và mọi số liệu downstream đều không đáng tin.
