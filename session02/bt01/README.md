# Bài 1: Phân tích & Lựa chọn — Cấu hình đa môi trường (Profiles)

## 1. Đáp án tối ưu nhất: **Phương án B**

```properties
# application-local.properties
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=qwen2.5-coder:7b

# application-cloud.properties
spring.ai.openai.api-key=${OPENROUTER_API_KEY}
spring.ai.openai.base-url=https://openrouter.ai/api/v1
spring.ai.openai.chat.options.model=google/gemini-2.5-flash

# application.properties
spring.profiles.active=local
```

### Lý do lựa chọn & Phân tích chuyên sâu:
- **Tuân thủ đúng cơ chế Spring Profiles & Quy chuẩn đặt tên:**  
  Spring Boot tự động load file cấu hình theo quy tắc `application-{profile}.properties`. Khi active profile nào (ví dụ `local` hoặc `cloud`), chỉ các thuộc tính của profile đó được nạp và kích hoạt.
- **Nguyên tắc đóng gói & Tách biệt môi trường (Separation of Concerns):**  
  Cấu hình môi trường local (chạy máy nội bộ) và cloud (chạy hạ tầng server/cloud) hoàn toàn độc lập. Giúp quản lý cấu hình rõ ràng, tránh rò rỉ hoặc ghi đè thuộc tính chéo giữa các môi trường.
- **Đảm bảo tính "Zero Code Change" (Không sửa mã nguồn Java):**  
  Tận dụng tối đa tính năng Auto-configuration của Spring AI. Khi chỉ nạp cấu hình của Ollama (ở profile `local`), Spring Boot chỉ khởi tạo duy nhất Bean `OllamaChatModel`. Khi chuyển sang profile `cloud`, Spring Boot khởi tạo Bean `OpenAiChatModel` mà **không cần viết thêm bất kỳ custom code hay `@Conditional` nào trong Java**.
- **Linh hoạt và an toàn khi triển khai (DevOps-friendly):**  
  - Khi chạy local: Developer không cần khai báo biến môi trường `${OPENROUTER_API_KEY}`. Ứng dụng vẫn khởi động trơn tru.
  - Khi deploy lên Staging/Production: Chỉ cần ghi đè profile qua biến môi trường hoặc tham số khởi chạy mà không cần sửa code/file cấu hình:
    ```bash
    java -jar app.jar --spring.profiles.active=cloud
    # hoặc
    export SPRING_PROFILES_ACTIVE=cloud
    ```

---

## 2. Phân tích nhược điểm & Lỗi kỹ thuật của các phương án còn lại

### ❌ Phương án A: Gộp chung toàn bộ cấu hình vào một file `application.properties`

- **Xung đột Bean (Bean Conflict / `NoUniqueBeanDefinitionException`):**  
  Cả cấu hình của Ollama và OpenAI đều được nạp đồng thời vào ApplicationContext. Nếu trong dự án có cả 2 dependency `spring-ai-ollama-spring-boot-starter` và `spring-ai-openai-spring-boot-starter`, cơ chế Auto-Configuration của Spring AI sẽ kích hoạt và khởi tạo đồng thời cả 2 bean implements interface `ChatModel` (`OllamaChatModel` và `OpenAiChatModel`).  
  Khi một Service gọi inject `@Autowired private ChatModel chatModel;` hoặc inject `ChatClient.Builder`, Spring Boot sẽ **báo lỗi khởi động** do phát hiện nhiều hơn một Bean cùng kiểu dữ liệu (`NoUniqueBeanDefinitionException: expected single matching bean but found 2`).
- **Lỗi phụ thuộc biến môi trường không cần thiết (Fail to start):**  
  Biến `${OPENROUTER_API_KEY}` nằm trực tiếp trong file dùng chung. Khi lập trình viên chạy local (chỉ muốn dùng Ollama offline), nếu trong máy không cấu hình biến môi trường `OPENROUTER_API_KEY`, ứng dụng sẽ văng lỗi `IllegalArgumentException: Could not resolve placeholder 'OPENROUTER_API_KEY'` ngay khi khởi động.
- **Không tận dụng được cơ chế Spring Profiles:**  
  Mặc dù có dòng `spring.profiles.active=local`, nhưng việc đặt toàn bộ properties vào file gốc `application.properties` khiến việc kích hoạt profile này trở nên vô nghĩa đối với các cấu hình bên dưới.

---

### ❌ Phương án C: Tự thiết lập key cấu hình riêng (Custom properties)

- **Sai quy chuẩn (Convention) của Spring AI:**  
  Spring AI sử dụng chuẩn tiền tố cấu hình chính thức (Official Properties Prefix) do đội ngũ Spring định nghĩa:
  - Ollama: `spring.ai.ollama.base-url`, `spring.ai.ollama.chat.options.model`,...
  - OpenAI: `spring.ai.openai.base-url`, `spring.ai.openai.api-key`, `spring.ai.openai.chat.options.model`,...  
  Các key tự chế như `spring.ai.active-model-type`, `spring.ai.ollama.url`, `spring.ai.openai.url` **hoàn toàn bị Spring AI Auto-Configuration bỏ qua**.
- **Vi phạm yêu cầu "Không sửa đổi bất kỳ dòng mã nguồn Java nào":**  
  Vì Spring AI không hiểu các key tự chế này, ứng dụng sẽ không thể tự tạo Bean `ChatModel`. Lập trình viên bắt buộc phải viết mã nguồn Java thủ công: tạo class `@ConfigurationProperties`, viết các hàm `@Bean`, sử dụng `@ConditionalOnProperty(name = "spring.ai.active-model-type", havingValue = "ollama")` hoặc tự khởi tạo `OllamaApi`/`OpenAiApi` bằng Java code. Điều này đi ngược lại hoàn toàn với tiêu chí đề bài.
- **Thiếu cấu hình bắt buộc:**  
  Cấu hình thiếu hoàn toàn các thông số quan trọng như API Key cho OpenAI (`api-key`) và tên mô hình cần gọi (`model`), dẫn đến model không đủ dữ liệu để vận hành.

---

## 3. Bảng tổng hợp so sánh

| Tiêu chí | Phương án A | Phương án B (Tối ưu) | Phương án C |
| :--- | :---: | :---: | :---: |
| **Tính chuẩn hóa (Standard Spring AI)** | Đúng key | Đúng key & Đúng chuẩn Profiles | ❌ Sai key chuẩn |
| **Không sửa code Java (Zero Code)** | ❌ Cần sửa code để giải quyết xung đột Bean | Tự động nạp đúng Bean theo Profile | ❌ Phải viết thêm class Config / Bean |
| **Tránh lỗi khởi động (Missing ENV)** | ❌ Bị lỗi nếu thiếu `OPENROUTER_API_KEY` ở local | Chỉ yêu cầu key khi chạy profile `cloud` | ❌ Thiếu cấu hình |
| **Khả năng mở rộng & Bảo trì** | Kém (Dễ rối khi thêm profile) | **Tốt nhất (Rõ ràng, độc lập, bảo mật)** | Kém (Phải tự duy trì boilerplate code) |
