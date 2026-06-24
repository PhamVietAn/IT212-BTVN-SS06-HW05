# BÀI 5: Thực hành Thiết kế Hệ thống Hướng Sự kiện và Xử lý Đơn hàng (Creative/Advanced)

## 1. Mô tả ý đồ thiết kế quy trình 3 bước tương tác liên tiếp

Quy trình thiết kế hệ thống hướng sự kiện (Event-Driven Architecture) cho SpeedyCart được triển khai thông qua một chuỗi 3 bước tương tác logic, đi từ tư vấn công nghệ vĩ mô đến thiết kế kịch bản chịu lỗi chi tiết và cuối cùng là sinh mã nguồn thực thi:

*   **Bước 1 - Tư vấn và lựa chọn Broker (Options & Trade-offs):**
    *   *Ý đồ:* Xác định "xương sống" cho kiến trúc hướng sự kiện. So sánh hai ứng cử viên hàng đầu là Apache Kafka và RabbitMQ để đưa ra lựa chọn phù hợp nhất dựa trên tính chất nghiệp vụ của SpeedyCart (quy mô dữ liệu lớn, cần mở rộng tốt, độ tin cậy cao).
*   **Bước 2 - Phân tích kịch bản sập hệ thống (What-if Scenario & Idempotency):**
    *   *Ý đồ:* Dự phòng rủi ro thực tế khi một Service tiêu thụ (Email Service) bị sập nguồn kéo dài (3 tiếng). Phân tích khả năng lưu trữ tạm của Broker và đưa ra giải pháp kiến trúc giải quyết bài toán "gửi trùng lặp" (Idempotency) khi phục hồi hệ thống. Sử dụng Redis làm Idempotency Key Store là một giải pháp chuẩn hóa trong hệ thống phân tán để loại bỏ các xử lý trùng lặp (Duplicate message consumption).
*   **Bước 3 - Thiết kế Prompt sinh mã nguồn CoT (Code Generation with CoT):**
    *   *Ý đồ:* Chuyển hóa thiết kế lý thuyết ở Bước 1 và Bước 2 thành mã nguồn Java Spring Boot chuẩn doanh nghiệp. Sử dụng kỹ thuật Chain-of-thought (CoT) để hướng dẫn AI viết code từng bước một cách chặt chẽ: kiểm tra trùng lặp qua Redis, gọi gửi mail, xử lý tự động gửi lại (Retry) khi mail server lỗi, và cuối cùng chuyển tin nhắn lỗi sang Dead Letter Queue (DLQ) để tránh làm nghẽn dòng chảy tin nhắn (Poison Pill handling).

---

## 2. Chi tiết cuộc hội thoại 3 bước chạy thực tế với AI

### LƯỢT CHAT 1: SO SÁNH VÀ LỰA CHỌN MESSAGE BROKER

#### Nội dung Prompt 1 thiết kế:
```markdown
[ROLE]
Bạn là một System Architect (Kiến trúc sư hệ thống) chuyên về các hệ thống hướng sự kiện (Event-Driven Architecture) chịu tải cao và có độ trễ cực thấp.

[GOAL]
Tư vấn và lập bảng so sánh chi tiết giữa hai giải pháp Message Broker: Apache Kafka và RabbitMQ để phân phối sự kiện `OrderCompletedEvent` trên SpeedyCart.

[CONTEXT]
Hệ thống SpeedyCart cần thực hiện các hành động phụ khi đơn hàng hoàn tất: gửi email xác nhận cho khách hàng, cộng điểm tích lũy thành viên, và thông báo cho bộ phận nhà kho đóng gói hàng. Để API phản hồi nhanh nhất và đảm bảo các dịch vụ hoạt động độc lập, hệ thống được thiết kế theo kiến trúc hướng sự kiện. Chúng tôi cần lựa chọn một Broker phù hợp nhất để chuyển tiếp sự kiện `OrderCompletedEvent` này từ Order Service sang các consumer độc lập (Email Service, Member Service, Warehouse Service).

[CONSTRAINTS]
Hãy tư vấn theo cấu trúc sau:
1. Đề xuất mô hình hoạt động của Apache Kafka và RabbitMQ đối với bài toán phân phối sự kiện này.
2. Lập bảng so sánh chi tiết giữa Apache Kafka và RabbitMQ dựa trên các tiêu chí cụ thể: Hiệu năng (Throughput & Latency), Khả năng mở rộng (Scalability), Độ tin cậy (Durability & Reliability), và Cơ chế phân phối tin nhắn (Message Routing & Delivery Guarantees).
3. Đưa ra khuyến nghị cuối cùng: Dự án SpeedyCart nên chọn công nghệ nào? Giải thích lý do cụ thể.

[FORMAT]
Trình bày khoa học, rõ ràng. Sử dụng bảng Markdown cho phần so sánh.
```

#### Phản hồi của AI (Lượt 1):
```markdown
Chào bạn, tôi là Kiến trúc sư hệ thống. Dưới đây là phân tích chi tiết và bảng so sánh giữa Apache Kafka và RabbitMQ cho hệ thống SpeedyCart của bạn.

### 1. Mô hình hoạt động đối với sự kiện OrderCompletedEvent

*   **RabbitMQ (Mô hình Smart Broker / Dumb Consumer):**
    *   Hoạt động theo mô hình AMQP. Khi `OrderCompletedEvent` được gửi lên, nó đi qua một **Exchange** (kiểu Fanout). Exchange này sẽ sao chép tin nhắn và đẩy vào 3 hàng đợi (Queue) riêng biệt: `email-queue`, `loyalty-queue`, và `warehouse-queue`.
    *   Mỗi Consumer (Email Service, Member Service, Warehouse Service) sẽ đăng ký lắng nghe hàng đợi tương ứng của mình. RabbitMQ sẽ chủ động đẩy (Push) tin nhắn xuống consumer và xóa tin nhắn khỏi hàng đợi ngay khi consumer xác nhận xử lý thành công (ACK).
*   **Apache Kafka (Mô hình Dumb Broker / Smart Consumer):**
    *   Hoạt động theo mô hình Log Append-Only phân tán. Sự kiện `OrderCompletedEvent` được ghi vào một **Topic** duy nhất tên là `order-completed`. Topic này có thể được chia làm nhiều Partition để tăng tải.
    *   Các consumer độc lập được tổ chức thành các **Consumer Group** khác nhau (ví dụ: `group-email`, `group-loyalty`, `group-warehouse`). Mỗi group tự duy trì một con trỏ chỉ vị trí đọc tin nhắn gọi là **Offset**. Kafka không xóa tin nhắn sau khi đọc; tin nhắn được lưu trữ trên đĩa cứng theo thời gian cấu hình (Retention Time, ví dụ: 7 ngày).

---

### 2. Bảng so sánh chi tiết Apache Kafka vs RabbitMQ

| Tiêu chí | RabbitMQ (Message Queue) | Apache Kafka (Event Streaming Platform) |
| :--- | :--- | :--- |
| **Hiệu năng (Throughput)** | **Trung bình - Cao** (~ hàng chục nghìn tin nhắn/giây). RAM được tận dụng tối đa để lưu trữ đệm tin nhắn. | **Cực kỳ cao** (hàng trăm nghìn đến hàng triệu tin nhắn/giây) nhờ cơ chế ghi tuần tự vào đĩa cứng (Sequential I/O) và Zero-Copy transfer. |
| **Độ trễ (Latency)** | **Rất thấp (< 2ms)**. Phù hợp cho xử lý thời gian thực (Real-time). | **Thấp (~ 5 - 10ms)**. Tối ưu cho xử lý theo lô (batching) để đạt băng thông tối đa. |
| **Khả năng mở rộng (Scalability)** | Khó mở rộng hơn. Khi dữ liệu trong Queue quá nhiều, RAM bị cạn kiệt và hiệu năng RabbitMQ giảm mạnh. | **Rất dễ mở rộng (Horizontal Scalability)** bằng cách thêm phân vùng (Partition) và nút máy chủ (Broker nodes) mà không làm suy giảm hiệu năng. |
| **Độ tin cậy (Durability)** | Hỗ trợ lưu trữ xuống đĩa cứng (Persistent Messages) nhưng làm giảm tốc độ xử lý của Broker. | **Mặc định lưu xuống đĩa** dưới dạng các commit log phân tán và nhân bản (Replication) giữa các nút trong cụm, đảm bảo an toàn tuyệt đối. |
| **Khả năng đọc lại dữ liệu (Replayability)** | **Không hỗ trợ.** Tin nhắn bị xóa vĩnh viễn khỏi hàng đợi ngay khi consumer gửi ACK thành công. | **Hỗ trợ rất tốt.** Consumer có thể kéo ngược con trỏ Offset để đọc lại và xử lý lại dữ liệu cũ bất cứ lúc nào trong thời gian lưu trữ. |
| **Định tuyến tin nhắn (Routing)** | **Cực kỳ linh hoạt** với các loại Exchange: Direct, Fanout, Topic, Headers cho phép định tuyến tin nhắn phức tạp. | **Cơ bản.** Định tuyến dựa trên khóa (Key-based routing) để phân bổ vào các Partition cụ thể. |

---

### 3. Khuyến nghị cho SpeedyCart

Đối với nghiệp vụ xử lý đơn hàng hoàn tất của SpeedyCart, **Apache Kafka là sự lựa chọn tối ưu hơn** nếu hệ thống định hướng mở rộng lớn (scale-up) trong tương lai. Tuy nhiên, nếu quy mô hiện tại ở mức vừa phải và yêu cầu tích hợp nhanh, **RabbitMQ cũng hoàn toàn đáp ứng tốt**.

**Lý do chọn Apache Kafka:**
1.  **Mô hình Publish/Subscribe hoàn hảo:** Một sự kiện `OrderCompletedEvent` được phát ra trên một Topic duy nhất và được đọc bởi nhiều dịch vụ khác nhau một cách hoàn toàn độc lập mà không cần nhân bản tin nhắn ở phía Broker như Exchange Fanout của RabbitMQ.
2.  **Khả năng lưu trữ và Replay dữ liệu:** Nếu một dịch vụ như Email Service bị sập, khi khởi động lại nó có thể đọc lại dữ liệu từ vị trí Offset cũ mà không sợ mất mát tin nhắn.
3.  **Khả năng chịu tải vượt trội:** Khi có các chương trình khuyến mãi lớn (Mega Sale), số lượng đơn hàng hoàn tất tăng đột biến, Kafka có thể chịu được lượng tải khổng lồ này mà không bị nghẽn RAM như RabbitMQ.
```

---

### LƯỢT CHAT 2: PHÂN TÍCH KỊCH BẢN SẬP HỆ THỐNG VÀ THIẾT KẾ CHỐNG TRÙNG LẶP (IDEMPOTENCY)

#### Nội dung Prompt 2 thiết kế:
```markdown
[ROLE]
Bạn là một Senior System Designer chuyên trách về giải pháp chịu lỗi (Fault-Tolerance) và tính toàn vẹn dữ liệu trong hệ thống phân tán.

[GOAL]
Hãy giải quyết bài toán chống trùng lặp tin nhắn (Idempotency) khi Email Service hoạt động trở lại sau sự cố sập nguồn 3 tiếng bằng giải pháp sử dụng Redis.

[CONTEXT]
Chúng tôi quyết định chọn Apache Kafka làm Message Broker cho SpeedyCart. Bây giờ, giả định một kịch bản sự cố thực tế xảy ra: Dịch vụ gửi Email (Email Service) bị sập nguồn đột ngột trong 3 tiếng, trong khi hệ thống đặt hàng và Kafka Broker vẫn hoạt động bình thường. 

[CONSTRAINTS]
Hãy trả lời các câu hỏi sau một cách chi tiết và có chiều sâu kỹ thuật:
1. Trong 3 tiếng Email Service bị sập, các tin nhắn chứa sự kiện đơn hàng (`OrderCompletedEvent`) trên Kafka Broker có bị mất hay không? Tại sao?
2. Khi Email Service hoạt động trở lại, nó sẽ nhận cùng một lúc hàng chục nghìn tin nhắn tích lũy. Làm thế nào để đảm bảo Email Service chỉ gửi đúng 1 email cho mỗi đơn hàng (chống gửi trùng email - Idempotency)?
3. Thiết kế giải pháp sử dụng Redis làm Idempotency Key Store:
   - Mô tả thuật toán/luồng xử lý chi tiết từ lúc consumer nhận tin nhắn đến lúc kiểm tra qua Redis, gửi mail, lưu trạng thái và xác nhận (ACK).
   - Chọn kiểu dữ liệu gì trên Redis để tối ưu việc lưu trữ? Cấu hình TTL (Time-to-Live) bao lâu là hợp lý? Tại sao?

[FORMAT]
Hãy trình bày giải pháp bằng văn bản phân tích logic và vẽ một quy trình (hoặc sơ đồ ASCII/Mermaid) mô tả luồng xử lý tin nhắn của Consumer.
```

#### Phản hồi của AI (Lượt 2):
```markdown
Chào bạn, dưới đây là giải pháp thiết kế kỹ thuật chi tiết xử lý kịch bản Email Service sập nguồn và cơ chế chống trùng lặp sử dụng Redis.

### 1. Tin nhắn trên Kafka Broker có bị mất không khi Email Service sập 3 tiếng?

**Trả lời:** **Không bị mất.**
*   **Lý do:** Khác với các Message Broker truyền thống xóa tin nhắn ngay khi đọc, Apache Kafka lưu trữ tất cả các tin nhắn trực tiếp xuống đĩa cứng vật lý dưới dạng Commit Log. Dữ liệu này chỉ bị xóa dựa trên cấu hình **Retention Policy** của Topic (mặc định của Kafka là 7 ngày).
*   Trong thời gian Email Service sập 3 tiếng, các tin nhắn đơn hàng hoàn tất vẫn được ghi đều đặn vào Topic `order-completed`. Vị trí đọc tin nhắn (Offset) của `group-email` sẽ dừng lại tại vị trí cuối cùng trước khi sập. Khi Service sống lại, nó sẽ tiếp tục kéo (Poll) dữ liệu từ điểm Offset đó trở đi.

---

### 2. Thiết kế cơ chế chống trùng lặp (Idempotency) sử dụng Redis

Khi Email Service sống lại, do cơ chế gửi tin nhắn của Kafka đảm bảo phân phối **At-Least-Once** (Ít nhất một lần) kết hợp với các khả năng xảy ra lỗi mạng trong quá trình xử lý, việc tin nhắn bị tiêu thụ lặp lại (Duplicate Consumption) là chắc chắn xảy ra. Ta cần thiết kế giải pháp Idempotency.

#### Sơ đồ luồng xử lý (ASCII Flowchart):

```
       [Kafka Broker]
             |
             | (1. Poll Message: Event ID = UUID)
             v
   +--------------------+
   |   Email Consumer   |
   +--------------------+
             |
             | (2. Check & Set: SETNX key "PROCESSING" EX 300)
             v
     +---------------+          Key Exists?
     |  Redis Cache  | ----------------------------+
     +---------------+                             |
             | Key Created (New Event)             | Yes (Duplicate or Processing)
             v                                     v
   +--------------------+                +--------------------+
   |  Send Email via    |                | Log Warning & ACK  |
   |  SMTP Mail Server  |                |   (Skip Message)   |
   +--------------------+                +--------------------+
             |
             |--- Success?
             |
             +---> Yes: (3. Update Key to "PROCESSED" EX 86400) ---> [Send ACK to Kafka]
             |
             +---> No (SMTP Error): (4. Delete Key from Redis)   ---> [Trigger Retry / DLQ]
```

#### Mô tả thuật toán chi tiết:
1.  **Consumer nhận tin nhắn:** Consumer đọc sự kiện `OrderCompletedEvent` chứa trường thông tin duy nhất là `eventId` (hoặc `orderId` - khuyến nghị dùng `eventId` kiểu UUID được sinh ra ngay khi tạo sự kiện).
2.  **Kiểm tra và chiếm khóa trên Redis (Idempotency Check):**
    *   Consumer thực hiện lệnh `SETNX` (Set if Not Exists) với key là `idempotency:email:{eventId}` và value là `"PROCESSING"`, đặt thời gian hết hạn tạm thời (ví dụ: 5 phút để tránh deadlock nếu consumer bị crash giữa chừng).
    *   **Trường hợp 1 (Key đã tồn tại):** Lệnh `SETNX` trả về `false`. Có nghĩa là sự kiện này đang được xử lý bởi một thread khác, hoặc đã xử lý thành công trước đó. Consumer lập tức dừng xử lý, ghi log cảnh báo và gửi ACK (xác nhận) cho Kafka để bỏ qua tin nhắn trùng lặp.
    *   **Trường hợp 2 (Key chưa tồn tại):** Lệnh `SETNX` trả về `true`. Consumer được phép đi tiếp vào bước gửi email.
3.  **Gửi Email:** Thực hiện gọi SMTP server để gửi email cho khách hàng.
4.  **Cập nhật trạng thái và ACK:**
    *   Nếu gửi email **thành công**: Cập nhật giá trị của Key trên Redis thành `"PROCESSED"` và cập nhật thời gian hết hạn (TTL) dài hơn (ví dụ: 24 giờ). Sau đó, thực hiện commit offset (ACK) gửi cho Kafka.
    *   Nếu gửi email **thất bại**: Xóa Key `idempotency:email:{eventId}` trên Redis để các lần thử lại sau (Retry) có cơ hội thực thi lại. Tiến hành cơ chế Retry/DLQ.

#### Lựa chọn kiểu dữ liệu và cấu hình TTL trên Redis:
*   **Kiểu dữ liệu:** Sử dụng kiểu dữ liệu **String** của Redis. Key là `idempotency:email:{eventId}`, Value là trạng thái (`PROCESSING` / `PROCESSED`). Kiểu dữ liệu này rất nhẹ, có thời gian truy xuất $O(1)$ và hỗ trợ trực tiếp lệnh nguyên tử `SETNX` kèm TTL.
*   **Cấu hình TTL (Time-To-Live):**
    *   Trạng thái `PROCESSING`: Đặt TTL là **5 phút (300 giây)**. Khoảng thời gian này đủ để Consumer thực hiện gửi email. Nếu quá 5 phút mà chưa xử lý xong (ví dụ Consumer bị sập giữa chừng), khóa tự động giải phóng để tin nhắn có thể được xử lý lại ở consumer khác.
    *   Trạng thái `PROCESSED`: Đặt TTL là **24 giờ (86400 giây)**. Đơn hàng hoàn tất và email xác nhận chỉ cần đảm bảo không gửi trùng trong vòng 1 ngày. Việc đặt TTL giúp Redis tự động dọn dẹp bộ nhớ RAM, tránh phình to dữ liệu theo thời gian.
```

---

### LƯỢT CHAT 3: SINH MÃ NGUỒN KAFKA CONSUMER TÍCH HỢP REDIS VÀ DLQ

#### Nội dung Prompt 3 thiết kế:
```markdown
[ROLE]
Bạn là một Senior Java Developer chuyên về Spring Cloud Stream, Kafka và Redis.

[GOAL]
Hãy viết mã nguồn Java Spring Boot hoàn chỉnh cho lớp `OrderEventListener` (Kafka Consumer) để lắng nghe sự kiện đơn hàng, tích hợp kiểm tra trùng lặp bằng Redis, cấu hình cơ chế tự động gửi lại (Retry) và chuyển sang Dead Letter Queue (DLQ) sau khi thất bại.

[CONSTRAINTS]
Yêu cầu mã nguồn phải tuân thủ nghiêm ngặt các quy tắc kỹ thuật sau:
1. Sử dụng chú thích `@KafkaListener` lắng nghe topic `order-completed`. Cấu hình consumer group là `group-email`.
2. Tích hợp với `StringRedisTemplate` để thực hiện thuật toán Idempotency chống trùng lặp gửi thư đã thiết kế ở Bước 2 sử dụng lệnh `opsForValue().setIfAbsent(...)` với TTL.
3. Khi nhận được tin nhắn, hãy in log INFO bắt đầu xử lý. Nếu kiểm tra trùng lặp phát hiện event đã được xử lý hoặc đang xử lý, hãy in log WARN và return sớm (skip).
4. Giả lập việc gọi `EmailService.sendConfirmationEmail(...)`. Bao bọc trong khối try-catch.
5. Nếu xảy ra lỗi kết nối với Mail Server:
   - Thực hiện cơ chế tự động gửi lại (Retry) 3 lần, mỗi lần cách nhau 2 giây. Bạn có thể sử dụng cấu hình retry của Spring Kafka (DefaultErrorHandler / RetryTemplate) hoặc tự code logic retry thủ công trong consumer.
   - Nếu sau 3 lần retry vẫn thất bại, hãy in log ERROR chi tiết, tiến hành xóa idempotency key trên Redis, và gửi tin nhắn lỗi sang một Dead Letter Queue (DLQ) có tên là `order-completed.DLQ` thông qua `KafkaTemplate` để các kỹ sư hệ thống kiểm tra thủ công sau.
6. Sử dụng `@Slf4j` và thư viện Lombok. Chú thích tiếng Việt đầy đủ trong mã nguồn.

[FORMAT]
Trả về mã nguồn Java hoàn chỉnh của hai lớp: `OrderEventListener` (Kafka Consumer) và `EmailService` (giả lập), cùng với cấu hình Spring Boot YAML mẫu cho Kafka Retry và DLQ nếu cần thiết.
```

#### Phản hồi của AI (Lượt 3):
```markdown
Chào bạn, tôi là Senior Developer. Dưới đây là mã nguồn Java Spring Boot hoàn chỉnh cho lớp Kafka Consumer (`OrderEventListener`) tích hợp Redis Idempotency Key Store, cơ chế Retry và xử lý Dead Letter Queue (DLQ).

### 1. Class Mock DTO Event `OrderCompletedEvent.java`:

```java
package com.speedycart.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.io.Serializable;
import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderCompletedEvent implements Serializable {
    private String eventId; // UUID duy nhất của sự kiện
    private String orderId;
    private String customerEmail;
    private String customerName;
    private BigDecimal totalAmount;
}
```

---

### 2. Class Mock Service gửi email `EmailService.java`:

```java
package com.speedycart.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class EmailService {

    /**
     * Giả lập gửi email xác nhận đơn hàng cho khách hàng.
     */
    public void sendConfirmationEmail(String email, String name, String orderId) throws Exception {
        log.info("Đang thực hiện gửi SMTP Email xác nhận cho đơn hàng: {} tới địa chỉ: {}", orderId, email);
        
        // Giả lập thỉnh thoảng gặp lỗi Mail Server để test cơ chế Retry
        if (Math.random() > 0.8) {
            throw new Exception("Lỗi kết nối Socket đến Mail Server SMTP (Timeout 504)");
        }
        
        log.info("Email đã được gửi thành công đến khách hàng: {}", name);
    }
}
```

---

### 3. Class Kafka Consumer `OrderEventListener.java` hoàn chỉnh:

```java
package com.speedycart.listener;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.speedycart.event.OrderCompletedEvent;
import com.speedycart.service.EmailService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventListener {

    private final StringRedisTemplate redisTemplate;
    private final EmailService emailService;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    private static final String IDEMPOTENCY_KEY_PREFIX = "idempotency:email:";
    private static final String DLQ_TOPIC = "order-completed.DLQ";

    /**
     * Lắng nghe sự kiện từ topic order-completed.
     */
    @KafkaListener(topics = "order-completed", groupId = "group-email")
    public void onOrderCompleted(String message) {
        OrderCompletedEvent event;
        try {
            event = objectMapper.readValue(message, OrderCompletedEvent.class);
        } catch (Exception e) {
            log.error("Không thể parse nội dung tin nhắn JSON. Lỗi: {}. Tin nhắn lỗi gửi thẳng sang DLQ.", e.getMessage());
            kafkaTemplate.send(DLQ_TOPIC, message);
            return;
        }

        String eventId = event.getEventId();
        String redisKey = IDEMPOTENCY_KEY_PREFIX + eventId;
        log.info("Nhận sự kiện OrderCompletedEvent. EventID: {}, OrderID: {}", eventId, event.getOrderId());

        // 1. Áp dụng cơ chế Idempotency với Redis bằng lệnh SETNX (setIfAbsent)
        // Set trạng thái "PROCESSING" hết hạn sau 5 phút (300 giây) phòng ngừa crash giữa chừng
        Boolean isNewEvent = redisTemplate.opsForValue().setIfAbsent(redisKey, "PROCESSING", Duration.ofMinutes(5));

        if (Boolean.FALSE.equals(isNewEvent)) {
            String currentStatus = redisTemplate.opsForValue().get(redisKey);
            log.warn("Sự kiện trùng lặp bị từ chối! EventID: {} đã được xử lý hoặc đang xử lý. Trạng thái hiện tại: {}", 
                    eventId, currentStatus);
            return; // Return sớm để tránh xử lý lại (Skip)
        }

        // 2. Tiến hành xử lý gửi email với cơ chế Retry thủ công
        boolean isSuccess = false;
        int maxRetries = 3;
        int attempt = 0;

        while (attempt < maxRetries && !isSuccess) {
            attempt++;
            try {
                emailService.sendConfirmationEmail(event.getCustomerEmail(), event.getCustomerName(), event.getOrderId());
                isSuccess = true;
            } catch (Exception e) {
                log.warn("Thử gửi email lần {}/{} thất bại cho đơn hàng {}. Lỗi: {}", 
                        attempt, maxRetries, event.getOrderId(), e.getMessage());
                if (attempt < maxRetries) {
                    try {
                        Thread.sleep(2000); // Đợi 2 giây trước khi thử lại
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                    }
                }
            }
        }

        // 3. Xử lý kết quả sau các lượt thử gửi mail
        if (isSuccess) {
            // Cập nhật trạng thái thành PROCESSED và kéo dài TTL lên 24 giờ để chống trùng lặp trong ngày
            redisTemplate.opsForValue().set(redisKey, "PROCESSED", Duration.ofHours(24));
            log.info("Xử lý thành công gửi email cho EventID: {}. Đã lưu trạng thái PROCESSED lên Redis.", eventId);
        } else {
            // Thất bại hoàn toàn sau 3 lần thử
            log.error("Gửi email thất bại hoàn toàn sau {} lần thử cho đơn hàng {}. Tiến hành xử lý lỗi.", maxRetries, event.getOrderId());
            
            // Xóa key trên Redis để có thể thực thi lại sau khi sửa xong hệ thống
            redisTemplate.delete(redisKey);
            
            // Đẩy tin nhắn lỗi sang Dead Letter Queue (DLQ) để phục vụ kiểm tra thủ công
            sendToDeadLetterQueue(event);
        }
    }

    /**
     * Gửi tin nhắn lỗi sang Dead Letter Queue (DLQ).
     */
    private void sendToDeadLetterQueue(OrderCompletedEvent event) {
        try {
            String jsonPayload = objectMapper.writeValueAsString(event);
            kafkaTemplate.send(DLQ_TOPIC, event.getEventId(), jsonPayload);
            log.info("Đã gửi tin nhắn lỗi của sự kiện {} sang Dead Letter Queue ({}) thành công.", 
                    event.getEventId(), DLQ_TOPIC);
        } catch (Exception e) {
            log.error("Không thể đẩy tin nhắn sang DLQ. Lỗi: {}", e.getMessage());
        }
    }
}
```

---

### 4. Cấu hình Spring Boot `application.yml` cho Kafka:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      group-id: group-email
      auto-offset-reset: earliest # Đọc từ offset cũ nhất nếu consumer bị sập nguồn
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
  data:
    redis:
      host: localhost
      port: 6379
```
```
