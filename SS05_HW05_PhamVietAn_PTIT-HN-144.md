# BÀI 5: Sáng tạo (Thiết kế Quy trình & Prompt cho Bộ Giới hạn Tần suất - Rate Limiter)

## 1. Mô tả ngắn gọn ý đồ thiết kế quy trình 3 bước
Hệ thống giới hạn tần suất (Rate Limiter) cho một API Gateway tài chính đòi hỏi tính chính xác, khả năng mở rộng phân tán và đặc biệt là độ tin cậy cực cao. Quy trình 3 bước tương tác với AI được thiết kế nhằm dẫn dắt AI đi từ tư duy kiến trúc vĩ mô đến lập trình chi tiết:

* **Bước 1 (Lựa chọn thuật toán):** Ép AI so sánh các thuật toán kinh điển (Token Bucket và Fixed Window). Việc này giúp nhóm phát triển hiểu rõ sự đánh đổi (Trade-off) về bộ nhớ và hành vi đột biến lưu lượng (Burst traffic) của từng thuật toán trước khi code.
* **Bước 2 (Giải quyết bài toán phân tán):** Đặt ra kịch bản thực tế khi hệ thống scale lên nhiều máy chủ (Distributed). Yêu cầu AI chuyển từ bộ nhớ local (Local memory) sang bộ lưu trữ tập trung Redis, phân tích các rủi ro đồng bộ và race conditions.
* **Bước 3 (Lập trình an toàn và dự phòng - Fallback):** Yêu cầu AI đóng vai Senior Developer để sinh code Java Spring Boot thực tế. Ràng buộc quan trọng ở bước này là **tính nguyên tử (Atomicity)** thông qua Redis Lua Script và **cơ chế dự phòng (Fallback)** khi Redis sập (Redis Down) - một kịch bản thực tế rất dễ xảy ra trên Production.

---

## 2. Nội dung của 3 Prompt tương ứng với các bước

### Prompt 1 (Bước 1 - Lựa chọn thuật toán):
```text
Tôi đang xây dựng một cổng API Gateway cho ứng dụng tài chính "SafePay" và cần thiết kế tính năng giới hạn tần suất truy cập (Rate Limiting). Quy tắc nghiệp vụ: Mỗi API Key chỉ được phép gửi tối đa 100 yêu cầu (requests) trong vòng 1 phút. Nếu vượt quá giới hạn, hệ thống trả về mã lỗi HTTP 429 (Too Many Requests).

Hãy đóng vai trò là một System Architect. Hãy giới thiệu và so sánh chi tiết ít nhất 2 thuật toán giới hạn tần suất phổ biến (Token Bucket và Fixed Window). Lập bảng so sánh ưu/nhược điểm và đánh giá xem thuật toán nào phù hợp hơn cho ứng dụng tài chính Java.
```

### Prompt 2 (Bước 2 - Thiết kế phân tán):
```text
Cảm ơn bạn. Bây giờ hãy phân tích kịch bản giả định sau (What-if scenario): "Nếu hệ thống SafePay chạy trên cụm 3 máy chủ (Distributed System), việc lưu trữ biến đếm số lượng requests trong bộ nhớ cục bộ (Local Memory) của từng server sẽ bị sai lệch do Load Balancer phân phối request ngẫu nhiên."

Hãy đề xuất giải pháp sử dụng Redis làm kho lưu trữ tập trung (Centralized Store) để giải quyết vấn đề trên. Giải thích cơ chế hoạt động, cấu trúc Key-Value lưu trên Redis và cách đảm bảo tính nguyên tử (Atomicity) khi nhiều server cùng đọc/ghi đồng thời vào một Key trên Redis.
```

### Prompt 3 (Bước 3 - Sinh mã nguồn CoT & Fallback):
```text
Hãy đóng vai trò là một Senior Java Developer. Hãy áp dụng cấu trúc suy nghĩ từng bước (CoT) để viết mã nguồn Java Spring Boot sử dụng Spring Data Redis nhằm triển khai bộ Rate Limiter phân tán này cho cổng API Gateway.

Yêu cầu cụ thể:
1. Viết một Spring HandlerInterceptor để chặn các request trước khi vào Controller và kiểm tra API Key từ Header "X-API-KEY".
2. Sử dụng Redis Lua Script để thực hiện việc tăng biến đếm và set TTL (60 giây) một cách nguyên tử (Atomic) nhằm tránh tranh chấp dữ liệu (Race Condition).
3. Thiết kế cơ chế dự phòng (Fallback) khi Redis đột ngột bị sập (ví dụ: Connection Timeout, sập cụm Redis). Giải thích lựa chọn của bạn giữa Fail-Open (cho qua và ghi log lỗi) và Fail-Closed (chặn hoàn toàn request) dưới góc độ của một hệ thống tài chính, sau đó viết code triển khai cơ chế đó.
4. Sử dụng Lombok và ghi log bằng @Slf4j. Chú thích tiếng Việt chi tiết.
```

---

## 3. Minh chứng thực tế: Log chat 3 lượt trực tiếp với AI

### [CHAT LOG - BƯỚC 1: TƯ VẤN & LỰA CHỌN THUẬT TOÁN]

#### Phản hồi từ AI:
Chào bạn, dưới đây là tư vấn từ góc độ System Architect về việc lựa chọn thuật toán Rate Limiting cho hệ thống SafePay:

Hai thuật toán phổ biến nhất để giải quyết bài toán này là **Fixed Window (Cửa sổ cố định)** và **Token Bucket (Thùng thẻ bài)**.

#### Bảng so sánh thuật toán Fixed Window và Token Bucket

| Tiêu chí | Thuật toán Cửa sổ cố định (Fixed Window) | Thuật toán Thùng thẻ bài (Token Bucket) |
| :--- | :--- | :--- |
| **Cơ chế hoạt động** | Chia thời gian thành các cửa sổ cố định (ví dụ: 1 phút). Mỗi API Key có một biến đếm (counter). Hết 1 phút, reset counter về 0. | Thùng chứa tối đa $N$ token. Mỗi giây tự động nạp thêm $r$ token vào thùng. Mỗi request thành công tiêu tốn 1 token. Nếu thùng rỗng, request bị chặn. |
| **Độ khó triển khai** | **Rất Dễ** (Chỉ cần tăng counter và set thời gian hết hạn cho key). | **Trung bình** (Phải tính toán số lượng token nạp thêm dựa trên khoảng thời gian giữa 2 requests). |
| **Mức tiêu thụ bộ nhớ** | **Rất ít** (Chỉ lưu một cặp Key-Counter). | **Ít** (Cần lưu Token count và Timestamp của request cuối cùng). |
| **Xử lý Burst Traffic** | **Kém**. Dễ bị lỗi "đột biến ở biên cửa sổ". Ví dụ: Client gửi 100 requests ở giây thứ 59 của phút thứ nhất, và 100 requests ở giây thứ 01 của phút thứ hai. Hệ thống phải chịu tải 200 requests trong vòng 2 giây (vi phạm quy tắc 100 req/phút). | **Tốt**. Cho phép xử lý đột biến lưu lượng (Burst traffic) trong giới hạn dung tích của thùng, sau đó tự động điều tiết về tốc độ nạp cố định. |

#### Đề xuất cho ứng dụng tài chính SafePay:
Đối với một cổng API Gateway tài chính, **Token Bucket** là thuật toán phù hợp hơn nhờ khả năng điều tiết lưu lượng mượt mà, tránh các đợt micro-burst làm quá tải hệ thống Core Banking phía sau. Tuy nhiên, nếu ưu tiên hàng đầu là hiệu năng xử lý cực nhanh và đơn giản tối đa để giảm thiểu độ trễ (latency) của Gateway, thuật toán **Sliding Window Counter** hoặc **Fixed Window** kết hợp Redis vẫn là sự lựa chọn phổ biến do tính dễ tối ưu hóa trên cụm phân tán.

---

### [CHAT LOG - BƯỚC 2: PHÂN TÍCH KỊCH BẢN GIẢ ĐỊNH PHÂN TÁN]

#### Phản hồi từ AI:
Trong kịch bản hệ thống chạy trên cụm 3 máy chủ phân tán đứng sau Load Balancer:

#### Tại sao bộ nhớ cục bộ (Local Memory) thất bại?
Nếu lưu trữ counter trên local memory (như Caffeine hay HashMap), thông tin request sẽ bị phân mảnh. Ví dụ: Client gửi 150 requests, Load Balancer chia đều mỗi server 50 requests. Cả 3 server đều thấy client chưa vượt quá hạn mức 100, nên đều cho qua. Kết quả: Client đã gửi thành công 150 requests (vượt hạn mức quy định), gây thất thoát tài nguyên hệ thống.

#### Giải pháp lưu trữ tập trung với Redis
Chúng ta sẽ chuyển toàn bộ trạng thái giới hạn tần suất lên **Redis** làm Database lưu trữ tập trung chia sẻ chung cho cả 3 máy chủ Spring Boot.

#### Cấu trúc Key-Value lưu trên Redis (Thuật toán Fixed Window):
- **Key:** `rate:limit:{apiKey}:{minute_timestamp}`
  - Trong đó: `apiKey` là mã định danh của client, `minute_timestamp` là timestamp của phút hiện tại (ví dụ: `202606221307` đại diện cho phút 13:07 ngày 22/06/2026).
- **Value:** Biến đếm (Integer) lưu số lượng requests đã gửi trong phút đó.
- **TTL (Time to Live):** 60 giây.

#### Đảm bảo tính nguyên tử (Atomicity)
Khi nhiều máy chủ Spring Boot cùng đọc/ghi đồng thời vào một key trên Redis, sẽ xảy ra lỗi tranh chấp dữ liệu (Race Condition). Ví dụ: Hai máy chủ cùng đọc thấy counter = 99, cả hai cùng quyết định cho qua và cùng tăng counter lên 100, dẫn đến thực tế có 101 requests được xử lý.
Để đảm bảo tính nguyên tử tuyệt đối, chúng ta sử dụng **Redis Lua Script**. Redis chạy đơn luồng (Single-threaded), khi một Lua Script được thực thi, Redis đảm bảo không có lệnh nào khác được xen vào cho đến khi script chạy xong. 

---

### [CHAT LOG - BƯỚC 3: SINH MÃ NGUỒN COT & FALLBACK LOGIC]

#### Phản hồi từ AI:
Chào bạn, dưới đây là quy trình suy nghĩ (CoT) và mã nguồn Java Spring Boot chi tiết cho bộ Rate Limiter phân tán.

#### Quy trình suy nghĩ (Chain-of-thought):
1. **Lựa chọn Interceptor:** Tôi chọn viết một `HandlerInterceptor` thay vì Filter vì nó tích hợp sâu vào Spring MVC, dễ dàng lấy cấu hình endpoint và bỏ qua các asset tĩnh.
2. **Thiết kế Lua Script:** Lua script cần nhận vào `Key` (rate limit) và `Limit` (100). Nó sẽ tự động tăng giá trị của key lên 1. Nếu giá trị mới bằng 1 (tức là key mới được tạo), script sẽ tự động đặt TTL là 60 giây. Trả về giá trị counter hiện tại.
3. **Phân tích chiến lược Fallback (Redis sập):**
   - **Fail-Closed (Chặn hoàn toàn):** An toàn cho hệ thống tài chính khỏi bị tấn công DDoS, nhưng sẽ làm sập toàn bộ dịch vụ nghiệp vụ (khách hàng không thể thanh toán được nữa).
   - **Fail-Open (Cho qua nhưng cảnh báo):** Bảo toàn trải nghiệm người dùng. Trong tài chính, việc gián đoạn thanh toán gây thiệt hại lớn hơn nhiều so với việc tạm thời cho phép vượt hạn mức rate limit.
   - *Quyết định:* Chọn **Fail-Open**. Nếu Redis sập, chúng ta sẽ bắt ngoại lệ, ghi log lỗi cảnh báo ở mức độ `WARN/ERROR` để đẩy cảnh báo về hệ thống giám sát (Prometheus/Grafana), và cho phép request đi tiếp vào Controller.
4. **Viết code:** Khởi tạo Spring Data Redis, load Lua Script, viết Interceptor chặn header "X-API-KEY".

Dưới đây là mã nguồn chi tiết:

#### 1. Lớp Cấu hình Redis: `RedisConfig.java`
```java
package com.safepay.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.util.List;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Sử dụng StringSerializer cho Key và Jackson cho Value
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    // Load Lua script từ thư mục resources
    @Bean
    @SuppressWarnings("unchecked")
    public RedisScript<Long> rateLimitLuaScript() {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setLocation(new ClassPathResource("scripts/rate_limit.lua"));
        script.setResultType(Long.class);
        return script;
    }
}
```

#### 2. File Lua Script: `src/main/resources/scripts/rate_limit.lua`
```lua
-- Nhận vào key cần đếm và giới hạn limit, thời gian hết hạn TTL
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local expire_time = tonumber(ARGV[2])

-- Tăng biến đếm lên 1
local current = redis.call('INCR', key)

-- Nếu là request đầu tiên trong phút này, đặt thời gian hết hạn (TTL = 60s)
if current == 1 then
    redis.call('EXPIRE', key, expire_time)
end

return current
```

#### 3. Bộ chặn kiểm tra Rate Limiter: `RateLimitInterceptor.java`
```java
package com.safepay.gateway.interceptor;

import com.safepay.gateway.config.RedisConfig;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Collections;
import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RedisTemplate<String, Object> redisTemplate;
    private final RedisScript<Long> rateLimitLuaScript;

    private static final int MAX_REQUEST_LIMIT = 100;
    private static final int WINDOW_SIZE_SECONDS = 60;
    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyyMMddHHmm");

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String apiKey = request.getHeader("X-API-KEY");

        // 1. Nếu không cung cấp API Key, từ chối ngay lập tức
        if (apiKey == null || apiKey.trim().isEmpty()) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("Missing X-API-KEY header");
            return false;
        }

        // Tạo Redis Key dựa trên API Key và Phút hiện tại
        String currentMinute = LocalDateTime.now().format(TIME_FORMATTER);
        String redisKey = String.format("rate:limit:%s:%s", apiKey, currentMinute);

        try {
            // 2. Gọi Lua Script một cách nguyên tử để kiểm tra và tăng biến đếm
            List<String> keys = Collections.singletonList(redisKey);
            Long currentRequestCount = redisTemplate.execute(
                    rateLimitLuaScript, 
                    keys, 
                    String.valueOf(MAX_REQUEST_LIMIT), 
                    String.valueOf(WINDOW_SIZE_SECONDS)
            );

            log.info("API Key: {} | Requests trong phút hiện tại: {}/{}", apiKey, currentRequestCount, MAX_REQUEST_LIMIT);

            // 3. Nếu vượt quá giới hạn, trả về mã HTTP 429
            if (currentRequestCount != null && currentRequestCount > MAX_REQUEST_LIMIT) {
                log.warn("API Key: {} đã vượt quá hạn mức truy cập! Trả về HTTP 429.", apiKey);
                response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value()); // HTTP 429
                response.setContentType("application/json");
                response.setCharacterEncoding("UTF-8");
                response.getWriter().write("{\"error\": \"Too Many Requests\", \"message\": \"Hạn mức của bạn là 100 requests/phút. Vui lòng thử lại sau.\"}");
                return false;
            }

        } catch (Exception e) {
            // 4. CƠ CHẾ DỰ PHÒNG (FALLBACK LOGIC) KHI REDIS SẬP
            // Áp dụng chiến lược "Fail-Open": Cho phép đi qua để đảm bảo tính liên tục của hệ thống tài chính
            // Ghi nhận log cảnh báo mức độ nghiêm trọng để bộ phận vận hành (SRE) phát hiện ngay lập tức
            log.error("HỆ THỐNG REDIS ĐỘT NGỘT GẶP SỰ CỐ! Kích hoạt cơ chế Fail-Open cho API Key: {}. Chi tiết lỗi: {}", 
                    apiKey, e.getMessage(), e);
            
            // Đặt Header để phía Client hoặc hệ thống giám sát biết đang chạy ở chế độ dự phòng
            response.setHeader("X-RateLimit-Fallback-Active", "true");
        }

        return true; // Cho phép đi tiếp vào Controller
    }
}
```

#### 4. Đăng ký Interceptor vào Spring: `WebMvcConfig.java`
```java
package com.safepay.gateway.config;

import com.safepay.gateway.interceptor.RateLimitInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final RateLimitInterceptor rateLimitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Áp dụng RateLimitInterceptor cho toàn bộ các API endpoint của SafePay Gateway
        registry.addInterceptor(rateLimitInterceptor)
                .addPathPatterns("/api/v1/**");
    }
}
```
```
