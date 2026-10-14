# BÀI 5: NỐI JOB TỰ ĐỘNG BUILD & PUSH DOCKER IMAGE SAU KHI TEST GRADLE XANH

- **Môn học:** DevOps / CI-CD
- **Bài thực hành:** Bài 5 - Session 11
- **Mục tiêu:** 
  - Tách luồng CI thành 2 Job riêng biệt: Job `test` (chạy Gradle test & đóng gói JAR) và Job `docker-build` (đóng gói Docker Image & Push lên GHCR).
  - Ràng buộc thứ tự thực thi bằng từ khóa `needs: test` để đảm bảo chỉ đóng gói Docker Image khi bài test Gradle thành công (tích xanh).
- **Đường dẫn nộp bài:** `homework/session_11/ex5/`

---

## 1. Cấu trúc thư mục dự án

```text
PTIT_CNTT1_IT209_Session11_Bai05/
├── .github/
│   └── workflows/
│       └── ci.yml                         # Kịch bản GitHub Actions có 2 Job nối tiếp nhau qua needs
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar             # Gradle wrapper executable JAR
│       └── gradle-wrapper.properties      # Cấu hình Gradle distribution (8.8)
├── src/
│   ├── main/java/com/example/demo/
│   │   └── DemoApplication.java          # Spring Boot Application entrypoint
│   └── test/java/com/example/demo/
│       └── DemoApplicationTests.java      # JUnit 5 Unit Tests
├── .dockerignore                          # Tối ưu hóa build context Docker
├── Dockerfile                             # Đóng gói ứng dụng Spring Boot từ file .jar
├── .gitignore                             # Loại trừ thư mục build, .gradle, IDE
├── build.gradle                           # Cấu hình Spring Boot 3.3.4, Java 17
├── gradlew                                # Script chạy trên Linux/macOS
├── gradlew.bat                            # Script chạy trên Windows
├── settings.gradle                        # Root project name
└── README.md                              # Tài liệu hướng dẫn & giải thích chi tiết
```

---

## 2. Kịch bản GitHub Actions Workflow (`.github/workflows/ci.yml`)

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

  docker-build:
    name: Build and Push Docker Image
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Tải thành phẩm JAR Artifact
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: build/libs

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - name: Thiết lập Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đóng gói và đẩy Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.actor }}/spring-boot-app:latest
```

---

## 3. Sơ đồ luồng thực thi (Workflow DAG)

Cấu hình `needs: test` tạo ra một đồ thị tuần tự (Directed Acyclic Graph) giữa các Job:

```mermaid
graph LR
    A["Job: Run Unit Tests (test)"] -->|needs: test (Success)| B["Job: Build and Push Docker Image (docker-build)"]
    
    style A fill:#2ea44f,stroke:#22863a,stroke-width:2px,color:#fff
    style B fill:#0366d6,stroke:#005cc5,stroke-width:2px,color:#fff
```

### Phân tích hành vi điều kiện:
1. **Trường hợp Test thành công (Success):**
   - Job `test` hoàn tất với mã thoát 0 (tất cả Unit test pass, build `.jar` thành công và tải lên Artifact).
   - Job `docker-build` được kích hoạt ngay lập tức, tải tệp `.jar` về, đóng gói Docker Image và push lên `ghcr.io`.
2. **Trường hợp Test thất bại (Failure):**
   - Nếu bất kỳ bài kiểm thử JUnit nào thất bại hoặc lệnh `./gradlew test` bị lỗi, Job `test` sẽ có trạng thái đỏ (Failed).
   - Job `docker-build` sẽ tự động chuyển sang trạng thái **Skipped** (Bị bỏ qua), ngăn chặn hoàn toàn việc build và phát hành một Docker Image chứa mã nguồn lỗi lên môi trường thực tế.

---

## 4. Tệp Dockerfile (`Dockerfile`)

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 5. Hướng dẫn kiểm tra và nộp bài

1. **Khởi tạo và đẩy code lên GitHub:**
   ```bash
   git init
   git add .
   git commit -m "feat: setup CI workflow with dependent jobs via needs"
   git branch -M main
   git remote add origin <URL_REPOSITORY_CỦA_BẠN>
   git push -u origin main
   ```
2. **Kiểm tra trên tab Actions:**
   - Quan sát biểu đồ Workflow (Workflow Graph): Có mũi tên nối từ Job **Run Unit Tests** sang Job **Build and Push Docker Image**.
   - Cả hai Job đều hoàn thành với trạng thái tích xanh.
3. **Thử nghiệm tính năng an toàn (Fail-safe):**
   - Sửa tệp `DemoApplicationTests.java` (ví dụ: đổi `assertEquals(30, a + b)` thành `assertEquals(99, a + b)`).
   - Đẩy commit lên GitHub: Job `test` sẽ báo đỏ (Failed) và Job `docker-build` sẽ hiển thị màu xám (Skipped).
4. **Chụp ảnh nộp bài:**
   - Ảnh chụp biểu đồ 2 Job kết nối thành công trên tab Actions của GitHub.
