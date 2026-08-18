# BÁO CÁO BÀI TẬP 2: TỐI ƯU PROMPT - TRÁNH BẪY THỜI GIAN TƯƠNG ĐỐI CHO AI AGENT

- **Học viên:** Dương Phúc Thịnh
- **Mã số sinh viên:** N24DTCN081
- **Môn học:** Kỹ năng ứng dụng AI
- **Đề tài:** Tối ưu hóa System Prompt và mã nguồn Java để xử lý thời gian động trong AI Agent đặt phòng

---

## PHẦN 1: TIÊU ĐỀ BÀI TẬP VÀ YÊU CẦU ĐỀ BÀI

### 1. Bối cảnh nghiệp vụ
Trợ lý ảo đặt phòng của hệ thống R-Hotels nhận được câu hỏi từ khách hàng: *"Tôi muốn tìm một phòng Deluxe từ ngày mai trong 3 ngày"*.

**Vấn đề lỗi:** Vì các mô hình ngôn ngữ lớn (LLM) hoạt động hoàn toàn tách biệt khỏi môi trường runtime của hệ thống backend, AI không có khái niệm về thời gian thực tế đang chạy. Nếu không được cung cấp mốc thời gian tham chiếu tĩnh cụ thể, AI sẽ:
1. Tự bịa ra một mốc thời gian ngẫu nhiên (gây ảo giác dữ liệu).
2. Truyền trực tiếp các chuỗi thời gian tương đối như `"ngày mai"`, `"tomorrow"`, `"3 ngày"` vào tham số của Tool/Function Call `getRoomAvailability`.

Hệ thống backend Java sử dụng kiểu dữ liệu chuẩn (`LocalDate` hoặc `Date`) để phân tích cú pháp. Khi nhận vào các chuỗi ngôn ngữ tự nhiên không đúng định dạng `yyyy-MM-dd`, hệ thống sẽ ném ra ngoại lệ `DateTimeParseException`, dẫn đến sập luồng API và trả về mã lỗi hệ thống nghiêm trọng HTTP 500.

### 2. Yêu cầu kỹ thuật
- **Thiết kế System Prompt động:** Có cấu trúc chặt chẽ gồm Vai trò (Role), Nhiệm vụ (Task), Ngữ cảnh (Context), Ràng buộc thời gian hệ thống (Time constraints) và Định dạng đầu ra tham số (Output formatting).
- **Chỉnh sửa mã nguồn Java REST Controller:** Lấy thời gian thực của máy chủ thông qua `LocalDate.now()`, tiêm động (inject) giá trị này vào System Prompt của mỗi phiên chat (Request-scoped) thay vì khởi tạo tĩnh một lần duy nhất tại Constructor.
- **Lập luận kỹ thuật:** Giải thích lý do tại sao cơ chế này triệt tiêu hoàn toàn lỗi crash hệ thống do lỗi parse định dạng ngày tháng.

---

## PHẦN 2: NỘI DUNG CUỘC TRÒ CHUYỆN THỰC TẾ VỚI AI

### 1. Câu lệnh Prompt gửi cho AI

```text
Chào bạn, tôi đang thiết kế một ứng dụng Spring AI cho hệ thống đặt phòng khách sạn R-Hotels. 
Hiện tại, mã nguồn Spring Boot REST Controller của tôi đang bị lỗi crash hệ thống (lỗi HTTP 500, ném ra ngoại lệ DateTimeParseException ở backend) khi người dùng nhập các cụm từ tương đối như "ngày mai", "3 ngày nữa". 

Dưới đây là đoạn mã nguồn hiện tại chưa tối ưu:

@RestController
@RequestMapping("/api/booking")
public class BookingController {
    private final ChatClient chatClient;
 
    public BookingController(ChatClient.Builder builder, BookingService bookingService) {
        this.chatClient = builder
                .defaultSystem("Bạn là trợ lý đặt phòng khách sạn. Hãy giúp khách hàng kiểm tra phòng trống.")
                .defaultTools(bookingService)
                .build();
    } 
 
    @GetMapping("/check")
    public String checkRoom(@RequestParam String message) {
        return this.chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
}

Hãy đóng vai trò là một kỹ sư AI xuất sắc, giúp tôi giải quyết bài toán này bằng cách:
1. Thiết kế lại một cấu trúc System Prompt động thật chặt chẽ (Vai trò, Nhiệm vụ, Ngữ cảnh, Ràng buộc thời gian, Định dạng đầu ra).
2. Chỉnh sửa và tối ưu hóa mã nguồn Java Controller để lấy mốc thời gian thực hiện tại của máy chủ qua LocalDate.now() và tiêm động vào System Prompt cho từng request.
3. Viết phân tích lập luận chi tiết vì sao giải pháp này khắc phục triệt để lỗi DateTimeParseException.
```

### 2. Phản hồi chi tiết từ AI

Sau đây là toàn bộ phản hồi giải pháp tối ưu được tạo ra bởi AI trợ lý học tập:

---

### KHUNG LÀM BÀI CỦA SINH VIÊN

#### 1. Thiết Kế System Prompt Động (Dynamic System Prompt)
Để đảm bảo AI luôn có mốc thời gian thực để đối chiếu và thực hiện phép toán cộng/trừ ngày tháng chuẩn xác, cấu trúc System Prompt dưới đây được thiết kế linh hoạt bằng cách sử dụng trình giữ chỗ `%s` cho biến thời gian máy chủ:

```text
[ROLE]
Bạn là Trợ lý ảo đặt phòng chuyên nghiệp và chính xác của chuỗi khách sạn R-Hotels.

[CONTEXT & TIME CONSTRAINT]
- Hôm nay là ngày (Thực tế): %s (Định dạng: yyyy-MM-dd).
- Đây là mốc thời gian toán học duy nhất để bạn tính toán tất cả các mốc thời gian tương đối mà người dùng cung cấp.
- TUYỆT ĐỐI không sử dụng bất kỳ mốc thời gian nào khác ngoài ngày hệ thống được cung cấp ở trên.

[TASK & REASONING RULES]
Khi người dùng cung cấp cụm từ chỉ thời gian tương đối, bạn phải thực hiện quy đổi logic logic theo các bước sau:
1. Xác định từ khóa tương đối: Ví dụ "hôm nay", "ngày mai", "ngày kia", "tuần sau", "trong X ngày".
2. Thực hiện phép toán ngày tháng dựa trên ngày tham chiếu ở trên:
   - "Hôm nay" = Ngày tham chiếu [%1$s]
   - "Ngày mai" = Ngày tham chiếu [%1$s] + 1 ngày
   - "Ngày kia" = Ngày tham chiếu [%1$s] + 2 ngày
   - "Trong X ngày" = Ngày bắt đầu + X ngày
3. Chuyển đổi toàn bộ kết quả tính toán về định dạng chuỗi chuẩn: "yyyy-MM-dd".

[TOOL CALL CONSTRAINT]
- Khi gọi công cụ (Tool/Function) `getRoomAvailability`, tham số truyền vào bắt buộc phải tuân thủ nghiêm ngặt định dạng chuỗi "yyyy-MM-dd" (Ví dụ: "2023-11-25").
- NGHIÊM CẤM truyền trực tiếp các cụm từ ngôn ngữ tự nhiên như "ngày mai", "tomorrow", "hôm nay", "3 ngày" hoặc các chuỗi định dạng sai lệch vào tham số của Tool.
- Nếu người dùng cung cấp thiếu thông tin thời gian, hãy lịch sự hỏi lại để làm rõ, không tự ý giả định.
```

---

#### 2. Mã Nguồn Java Controller Sau Khi Tối Ưu
Để khắc phục bẫy thời gian tương đối, mã nguồn Java cần được chuyển dịch từ việc cấu hình System Prompt tĩnh tại Constructor sang việc xây dựng cấu trúc System Prompt động tại thời điểm xử lý request (Runtime Request-scoped) bằng cách kết hợp `LocalDate.now()`.

```java
package com.rhotels.api.booking;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@RestController
@RequestMapping("/api/booking")
public class BookingController {
    
    private final ChatClient chatClient;
    
    // Bản mẫu System Prompt với placeholder %s để tiêm thời gian hệ thống động
    private static final String SYSTEM_PROMPT_TEMPLATE = """
            [ROLE]
            Bạn là Trợ lý ảo đặt phòng chuyên nghiệp và chính xác của chuỗi khách sạn R-Hotels.
            
            [CONTEXT & TIME CONSTRAINT]
            - Hôm nay là ngày (Thực tế): %s (Định dạng: yyyy-MM-dd).
            - Đây là mốc thời gian toán học duy nhất để bạn tính toán tất cả các mốc thời gian tương đối mà người dùng cung cấp.
            - TUYỆT ĐỐI không sử dụng bất kỳ mốc thời gian nào khác ngoài ngày hệ thống được cung cấp ở trên.
            
            [TASK & REASONING RULES]
            Khi người dùng cung cấp cụm từ chỉ thời gian tương đối, bạn phải thực hiện quy đổi logic:
            1. Xác định từ khóa tương đối: Ví dụ "hôm nay", "ngày mai", "ngày kia", "trong X ngày".
            2. Thực hiện phép toán ngày tháng dựa trên ngày tham chiếu ở trên:
               - "Hôm nay" = Ngày tham chiếu [%s]
               - "Ngày mai" = Ngày tham chiếu [%s] + 1 ngày
               - "Ngày kia" = Ngày tham chiếu [%s] + 2 ngày
               - "Trong X ngày" = Ngày bắt đầu + X ngày
            3. Chuyển đổi toàn bộ kết quả tính toán về định dạng chuỗi chuẩn: "yyyy-MM-dd".
            
            [TOOL CALL CONSTRAINT]
            - Khi gọi công cụ `getRoomAvailability`, tham số truyền vào bắt buộc phải tuân thủ nghiêm ngặt định dạng chuỗi "yyyy-MM-dd".
            - NGHIÊM CẤM truyền trực tiếp các cụm từ ngôn ngữ tự nhiên như "ngày mai", "tomorrow", "3 ngày" vào tham số của Tool.
            """;

    public BookingController(ChatClient.Builder builder, BookingService bookingService) {
        // Đăng ký công cụ nghiệp vụ vào ChatClient tại thời điểm khởi tạo
        this.chatClient = builder
                .defaultTools(bookingService)
                .build();
    }
 
    @GetMapping("/check")
    public String checkRoom(@RequestParam String message) {
        // Bước 1: Lấy ngày hiện tại thực tế trên máy chủ hệ thống
        LocalDate today = LocalDate.now();
        String todayStr = today.format(DateTimeFormatter.ISO_LOCAL_DATE); // Định dạng: yyyy-MM-dd
        
        // Bước 2: Tạo System Prompt hoàn chỉnh bằng cách tiêm động giá trị ngày hôm nay vào khuôn mẫu
        String dynamicSystemPrompt = String.format(SYSTEM_PROMPT_TEMPLATE, todayStr, todayStr, todayStr, todayStr);
        
        // Bước 3: Gửi request đến AI kèm System Prompt động vừa tạo
        return this.chatClient.prompt()
                .system(dynamicSystemPrompt) // Thiết lập System Prompt động cho từng luồng request cụ thể
                .user(message)
                .call()
                .content();
    }
}
```

---

#### 3. Phân Tích & Lập Luận Kỹ Thuật

Giải pháp này giải quyết triệt để lỗi logic của AI Agent thông qua các khía cạnh kỹ thuật sau:

##### A. Lỗi "Bẫy Thời Gian Tĩnh" trong mã nguồn cũ
Trong đoạn mã nguồn ban đầu:
```java
this.chatClient = builder.defaultSystem("...").defaultTools(bookingService).build();
```
Nếu lập trình viên ghi cứng ngày hệ thống trong `.defaultSystem()` hoặc khởi tạo ngày hệ thống trong Constructor của Spring Bean, giá trị ngày tháng đó chỉ được tính toán **một lần duy nhất** lúc máy chủ khởi động (Application Startup). 
- Nếu máy chủ chạy liên tục trong nhiều tháng, mốc thời gian tham chiếu của AI sẽ bị kẹt lại ở ngày khởi chạy ứng dụng.
- Việc định cấu hình System Prompt tĩnh không thể cung cấp khả năng tự tính toán theo thời gian thực mỗi ngày.

##### B. Cơ chế hoạt động của Giải pháp Tiêm Động (Dynamic Injection)
- **Request-scoped Injection:** Trong đoạn mã tối ưu hóa, phương thức `LocalDate.now()` được đặt bên trong endpoint `@GetMapping("/check")`. Điều này đảm bảo rằng với **mỗi request HTTP mới** được gửi từ người dùng, máy chủ sẽ chụp lại (snapshot) chính xác thời gian thực tế của hệ thống tại tích tắc đó.
- **Làm mốc toán học (Mathematical Anchor):** Bằng cách đưa giá trị `todayStr` vào trong System Prompt gửi đến mô hình LLM, ta đã thiết lập một "Neo thời gian". Mô hình ngôn ngữ lớn (LLM) không có đồng hồ bên trong, nhưng nó cực kỳ giỏi trong việc thực hiện các suy luận logic dựa trên ngữ cảnh được cung cấp (In-context learning). Khi biết ngày hôm nay là `2023-11-24`, nó có thể suy luận chính xác bằng toán học rằng "ngày mai" là `2023-11-25`.

##### C. Ngăn chặn lỗi `DateTimeParseException` (HTTP 500) ở Backend
- Khi không được định hướng, LLM sẽ cố gắng map chuỗi chữ viết tự nhiên `"ngày mai"` trực tiếp vào tham số `startDate` của Tool Call `getRoomAvailability(String startDate, int duration)`.
- Nhờ các ràng buộc nghiêm ngặt trong phần `[TOOL CALL CONSTRAINT]`, mô hình được cảnh báo rõ ràng về hậu quả của việc truyền sai định dạng chuỗi. 
- Hệ thống ràng buộc của Prompt biến đổi AI từ một công cụ sinh chữ ngẫu nhiên thành một bộ biên dịch thông minh: Nhận diện ngôn ngữ tự nhiên -> Tính toán toán học -> Trả về dữ liệu chuẩn cấu trúc định dạng. Tham số truyền vào Spring Boot backend được bảo mật tuyệt đối ở định dạng `yyyy-MM-dd`, triệt tiêu hoàn toàn khả năng ném ngoại lệ phân tích ngày tháng và đảm bảo tính ổn định bền vững của API.