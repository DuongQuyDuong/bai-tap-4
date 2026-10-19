# BÀI TẬP: ĐÓNG GÓI JAR & TÁI SỬ DỤNG THƯ VIỆN CALCULATOR

> Đây là bản **gộp mã nguồn** để xem/nộp GitHub; muốn chạy cần tách file đúng đường dẫn hoặc dùng ZIP dự án kèm theo.

## Kết quả kiểm thử bằng javac, jar và java
```text
Kết quả gọi từ thư viện đóng gói (.jar):
Tổng 2 số (5, 9): 14
Hiệu 2 số (5, 9): -4
Tích 2 số (5, 9): 45
Thương 2 số (10, 5): 2
```

## Mã nguồn từng file

### `math-lib/pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.codegym</groupId>
  <artifactId>math-lib</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
</project>
```

### `math-lib/src/main/java/com/codegym/Calculator.java`
```java
package com.codegym;

import java.security.NoSuchAlgorithmException;

public class Calculator {
    public static int sum(int a, int b) { return a + b; }
    public static int sub(int a, int b) { return a - b; }
    public static int mul(int a, int b) { return a * b; }

    public static int divide(int a, int b) throws NoSuchAlgorithmException {
        if (b == 0) {
            throw new NoSuchAlgorithmException("Không thể chia cho 0!");
        }
        return a / b;
    }
}
```

### `calculator-app/pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.codegym</groupId>
  <artifactId>calculator-app</artifactId>
  <version>1.0-SNAPSHOT</version>
  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
    <dependency>
      <groupId>com.codegym</groupId>
      <artifactId>math-lib</artifactId>
      <version>1.0-SNAPSHOT</version>
      <scope>system</scope>
      <systemPath>${project.basedir}/lib/math-lib.jar</systemPath>
    </dependency>
  </dependencies>
</project>
```

### `calculator-app/src/main/java/com/codegym/Main.java`
```java
package com.codegym;

import java.security.NoSuchAlgorithmException;

public class Main {
    public static void main(String[] args) {
        System.out.println("Kết quả gọi từ thư viện đóng gói (.jar):");
        System.out.println("Tổng 2 số (5, 9): " + Calculator.sum(5, 9));
        System.out.println("Hiệu 2 số (5, 9): " + Calculator.sub(5, 9));
        System.out.println("Tích 2 số (5, 9): " + Calculator.mul(5, 9));
        try {
            System.out.println("Thương 2 số (10, 5): " + Calculator.divide(10, 5));
            // Bỏ comment dòng dưới để thử ngoại lệ:
            // System.out.println(Calculator.divide(10, 0));
        } catch (NoSuchAlgorithmException e) {
            System.err.println("Lỗi thuật toán: Không thể chia cho 0!");
            e.printStackTrace();
        }
    }
}
```

### `README.md`
```markdown
# Bài thực hành đóng gói thư viện Calculator

Gồm `math-lib` (thư viện Java) và `calculator-app` (ứng dụng sử dụng thư viện).

## Chạy bằng Maven (Java 17+ và Maven)

1. Trong `math-lib`, chạy `mvn clean package`.
2. Copy `math-lib/target/math-lib-1.0-SNAPSHOT.jar` sang `calculator-app/lib/math-lib.jar` (đã có sẵn bản JAR được biên dịch trong ZIP).
3. Trong `calculator-app`, chạy `mvn clean package`.
4. **Windows** (PowerShell): `java -cp "target/classes;lib/math-lib.jar" com.codegym.Main`
5. **Linux/macOS**: `java -cp "target/classes:lib/math-lib.jar" com.codegym.Main`

Thử lỗi chia 0: bỏ comment ở `Main.java` rồi build/chạy lại.

**Ghi chú:** `scope=system` được sử dụng để bám sát yêu cầu bài thực hành; với dự án thực tế nên dùng `mvn install` và dependency thông thường. Phương thức `divide` dùng `NoSuchAlgorithmException` đúng theo đề, dù thực tế `ArithmeticException` phù hợp hơn.
```

## Các bước nộp GitHub
Upload file gộp `.md` hoặc giải nén ZIP rồi upload cả thư mục, giữ nguyên cấu trúc. Sau đó lấy URL GitHub gửi CodeGym.

**Lưu ý:** Đã kiểm thử việc biên dịch/chạy với `javac`, `jar`, `java` (Java 21, target 17). Chưa chạy Maven vì môi trường không có lệnh `mvn`.
