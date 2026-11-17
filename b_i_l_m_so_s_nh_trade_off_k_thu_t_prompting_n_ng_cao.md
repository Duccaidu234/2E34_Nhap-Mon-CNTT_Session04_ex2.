# BÀI TẬP THỰC HÀNH: SO SÁNH TRADE-OFF CÁC GIẢI PHÁP RA LỆNH CHO KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

## BỐI CẢNH BÀI TOÁN ỨNG DỤNG THỰC TẾ
Để minh họa rõ nét nhất cho việc tối ưu hóa quy trình **Chain-of-Thought (Chuỗi suy nghĩ)**, **Prompt Chaining (Nối chuỗi câu lệnh)** và **Xử lý đứt gãy dữ liệu (Exception/Fallback Handling)**, bài làm chọn một bài toán nghiệp vụ cụ thể:
* **Bài toán:** Xây dựng hệ thống AI tự động xử lý phản hồi khách hàng (Customer Support Triage & Resolution).
* **Quy trình chuỗi (Chaining):** 
  1. Đọc nội dung khiếu nại $\rightarrow$ Trích xuất thông tin đơn hàng (Mã đơn, Lỗi gặp phải).
  2. Phân tích mức độ nghiêm trọng (Logic CoT).
  3. Đề xuất phương án đền bù hoặc chuyển tiếp bộ phận kỹ thuật.
* **Bẫy dữ liệu / Điểm đứt gãy (Edge Cases):** Khách hàng cung cấp thiếu mã đơn hàng, nội dung khiếu nại mơ hồ/chứa ngôn từ cảm xúc cực đoan, hoặc lỗi nằm ngoài danh mục hỗ trợ tiêu chuẩn.

---

## BƯỚC 1: ĐỀ XUẤT 2 PROMPT MẪU

### Giải pháp 1: Phương án Đơn giản, Ngắn gọn (Zero-shot / Basic CoT)
Phương án này tập trung vào sự tự nhiên, ít từ khóa rườm rà, dựa vào khả năng suy luận mặc định của mô hình ngôn ngữ lớn (LLM).

> **Prompt Mẫu 1:**
> ```text
> Bạn là nhân viên chăm sóc khách hàng. Hãy đọc email khiếu nại dưới đây và xử lý theo từng bước:
> 1. Tóm tắt vấn đề và tìm mã đơn hàng.
> 2. Suy nghĩ từng bước (Chain-of-Thought) để đánh giá mức độ nghiêm trọng của lỗi.
> 3. Đưa ra phương án giải quyết phù hợp cho khách hàng.
> 
> Nếu thiếu thông tin thì hãy hỏi lại khách hàng.
> 
> Email khách hàng: "[Chèn nội dung email]"
> ```

---

### Giải pháp 2: Phương án Chi tiết, Chặt chẽ (Structured Prompt Chaining + Fallback Rules)
Phương án này áp dụng kỹ thuật phân tách luồng (Chaining), ép buộc định dạng đầu ra (JSON/XML) để chống đứt gãy khi truyền dữ liệu giữa các bước, và thiết lập các luật chặn lỗi biên (Guardrails) rõ ràng.

> **Prompt Mẫu 2:**
> ```text
> ### VAI TRÒ & MỤC TIÊU
> Bạn là Hệ thống Điều phối Khiếu nại Tự động (Triage AI). Nhiệm vụ của bạn là phân tích email, suy luận logic và xuất dữ liệu chuẩn hóa cho bước xử lý tiếp theo trong chuỗi (Pipeline).
> 
> ### QUY TRÌNH XỬ LÝ CHUỖI (PROMPT CHAINING & CoT)
> Thực hiện tuần tự 3 bước sau và hiển thị rõ quá trình suy luận trong thẻ <thinking>:
> - Bước 1 (Trích xuất): Tìm [Mã đơn hàng] (định dạng DH-XXXX) và [Loại lỗi].
> - Bước 2 (Đánh giá Logic - CoT): Dựa trên [Loại lỗi], phân loại mức độ:
>   + CAO: Lỗi thanh toán, mất hàng, hỏng hóc vật lý.
>   + TRUNG BÌNH: Giao chậm dưới 3 ngày, sai màu/kích thước.
>   + THẤP: Hỏi thông tin chung.
> - Bước 3 (Hành động): Nếu mức độ CAO -> Chuyển chuyên viên cấp 2. Nếu TRUNG BÌNH/THẤP -> Tạo mã giảm giá 10% xin lỗi.
> 
> ### QUY TẮC XỬ LÝ ĐỨT GÃY & BẪY DỮ LIỆU (FALLBACK RULES)
> 1. Bẫy thiếu dữ liệu: Nếu không tìm thấy [Mã đơn hàng] hợp lệ (DH-XXXX), DỪNG quy trình đánh giá đền bù, chuyển trạng thái sang "PENDING_INFO" và soạn câu hỏi yêu cầu bổ sung mã đơn.
> 2. Bẫy nhiễu thông tin: Bỏ qua các từ ngữ cảm xúc/chửi bới, chỉ tập trung vào sự kiện thực tế. Nếu email hoàn toàn vô nghĩa/spam, gán trạng thái "INVALID_INPUT".
> 
> ### ĐỊNH DẠNG ĐẦU RA BẮT BUỘC (JSON)
> Chỉ trả về duy nhất một khối JSON hợp lệ để hệ thống API đọc tiếp, không kèm văn bản thừa:
> {
>   "thinking_process": "Chuỗi suy luận từng bước của bạn",
>   "order_id": "String hoặc null",
>   "severity": "CAO | TRUNG BÌNH | THẤP | UNKNOWN",
>   "status": "RESOLVED | ESCALATED | PENDING_INFO | INVALID_INPUT",
>   "customer_reply": "Nội dung phản hồi khách hàng lịch sự"
> }
> 
> ### DỮ LIỆU ĐẦU VÀO
> Email: "[Chèn nội dung email]"
> ```

---

## BƯỚC 2: LẬP BẢNG SO SÁNH TRADE-OFF (ĐÁNH ĐỔI)

Dưới đây là bảng phân tích sự đánh đổi giữa hai phương án dựa trên các tiêu chí cốt lõi của kỹ thuật Prompt Engineering:

| Tiêu chí đánh giá | Giải pháp 1 (Đơn giản, ngắn gọn) | Giải pháp 2 (Chi tiết, nhiều ràng buộc) | Phân tích sự đánh đổi (Trade-off) |
| :--- | :--- | :--- | :--- |
| **Độ dễ hiểu (Readability)** | **Cao** <br>*(Ngôn ngữ tự nhiên, ai cũng có thể đọc, viết và chỉnh sửa nhanh chóng)* | **Trung bình** <br>*(Cấu trúc phức tạp, nhiều quy tắc lồng ghép, đòi hỏi người viết có tư duy hệ thống)* | Đánh đổi sự đơn giản khi bảo trì để lấy tính chính xác và kỷ luật của mô hình. |
| **Tốn Token (Chi phí & Độ trễ)** | **Ít** (~80 - 100 tokens) <br>*(Tiết kiệm chi phí API, tốc độ phản hồi nhanh)* | **Nhiều hơn** (~300 - 350 tokens) <br>*(Tốn chi phí đầu vào cao gấp 3-4 lần, độ trễ xử lý tăng nhẹ)* | Đánh đổi chi phí tính toán và độ trễ (latency) để giảm thiểu rủi ro sai sót nghiệp vụ. |
| **Chặn lỗi biên (Edge Cases)** | **Trung bình** <br>*(Dễ bị "ảo giác" hoặc tự ý bịa ra cách giải quyết khi gặp email thiếu thông tin)* | **Cực tốt** <br>*(Có cơ chế Fallback rõ ràng: gặp thiếu mã đơn hoặc spam sẽ tự động ngắt luồng an toàn)* | Giải pháp 1 dễ sập khi gặp dữ liệu bẩn; Giải pháp 2 miễn nhiễm tốt với các bẫy dữ liệu. |
| **Chống đứt gãy chuỗi (Chaining)** | **Kém** <br>*(Đầu ra là văn bản tự do, bước sau rất khó bóc tách dữ liệu tự động)* | **Xuất sắc** <br>*(Ép kiểu JSON cố định, đảm bảo bước 1 truyền dữ liệu mượt mà sang bước 2, 3)* | Giải pháp 1 chỉ hợp chat thủ công; Giải pháp 2 bắt buộc phải dùng nếu tích hợp vào hệ thống phần mềm. |
| **Tính nhất quán (Consistency)** | **Thấp** <br>*(Cùng 1 email nhưng chạy 5 lần có thể ra 5 kiểu trình bày khác nhau)* | **Rất cao** <br>*(100 lần chạy đều tuân thủ đúng 1 khung logic và 1 cấu trúc đầu ra)* | Đánh đổi sự linh hoạt sáng tạo của AI để lấy tính ổn định chuẩn công nghiệp. |

---

## BƯỚC 3: LỰA CHỌN GIẢI PHÁP TỐI ƯU VÀ CHẠY THỬ NGHIỆM

### 1. Lựa chọn phương án & Lý do
* **Phương án được chọn:** **Giải pháp 2 (Chi tiết, có nhiều ràng buộc)**.
* **Lý do lựa chọn:**
  1. **Đặc thù của kỹ thuật Chuỗi (Prompt Chaining):** Bản chất của *Chaining* là đầu ra của bước trước làm đầu vào cho bước sau. Nếu dùng Giải pháp 1, văn bản trả về lộn xộn sẽ gây **đứt gãy hệ thống** (hệ thống không thể tự động đọc được mức độ nghiêm trọng hay mã đơn hàng). Giải pháp 2 giải quyết triệt để bằng cấu trúc JSON và thẻ `<thinking>`.
  2. **Kiểm soát rủi ro nghiệp vụ:** Trong thực tế, dữ liệu người dùng luôn chứa "bẫy" (thiếu thông tin, spam, cảm xúc hỗn loạn). Chi phí bỏ thêm cho ~250 tokens ở Giải pháp 2 là **rất rẻ** so với thiệt hại khi AI tự ý hứa đền bù sai quy định hoặc bỏ sót khiếu nại nghiêm trọng.

---

### 2. Chạy thử nghiệm thực tế (Simulation) với Bẫy dữ liệu

Để kiểm chứng khả năng **xử lý đứt gãy và chặn lỗi biên**, ta đưa vào một email chứa nhiều bẫy: *Khách hàng rất tức giận, viết sai chính tả, không cung cấp mã đơn hàng theo chuẩn DH-XXXX, và đòi đền bù ngay lập tức.*

* **Dữ liệu đầu vào (Input Email):**
  > *"Làm ăn kiểu gì thế hả?? Tôi đặt cái áo từ tuần trước mà giờ vẫn chưa thấy đâu, gọi tổng đài thì tò te tí. Bực hết cả mình! Đền tiền cho tôi ngay lập tức không tôi bóc phốt lên mạng xã hội bây giờ!!"*

* **Kết quả chạy thử bằng Giải pháp 2 (Output):**

```json
{
  "thinking_process": "Bước 1: Quét toàn bộ email để tìm mã đơn hàng định dạng DH-XXXX -> Không tìm thấy. Loại lỗi: Giao hàng chậm trễ quá 1 tuần. Bước 2: Kích hoạt quy tắc xử lý đứt gãy số 1 (Thiếu mã đơn hàng) và quy tắc số 2 (Bỏ qua thái độ tức giận/đe dọa bóc phốt). Bước 3: Dừng luồng đánh giá đền bù vì thiếu dữ liệu định danh đơn hàng, chuyển trạng thái sang PENDING_INFO để yêu cầu khách cung cấp mã đơn.",
  "order_id": null,
  "severity": "UNKNOWN",
  "status": "PENDING_INFO",
  "customer_reply": "Chào bạn, chúng tôi thành thật xin lỗi về trải nghiệm không tốt và sự chậm trễ trong quá trình giao hàng khiến bạn phiền lòng. Để bộ phận hỗ trợ có thể kiểm tra ngay lập tức hành trình đơn hàng và có phương án xử lý thỏa đáng nhất cho bạn, bạn vui lòng cung cấp giúp chúng tôi Mã đơn hàng (có định dạng DH-XXXX) nhé. Rất mong nhận được phản hồi từ bạn!"
}
```

### 3. Đánh giá kết quả chạy thử
* **Không bị bẫy cảm xúc:** AI không hoảng loạn trước lời đe dọa "bóc phốt" mà tự ý duyệt đền bù (lỗi thường gặp ở Giải pháp 1).
* **Xử lý đứt gãy hoàn hảo:** Khi phát hiện `order_id` bị `null`, luồng suy luận lập tức rẽ nhánh an toàn sang `PENDING_INFO`, giữ nguyên được định dạng JSON chuẩn xác để hệ thống tự động gửi email hỏi lại khách hàng mà không làm sập quy trình chung.