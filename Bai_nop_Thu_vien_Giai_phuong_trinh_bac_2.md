# BÀI NỘP: ĐÓNG GÓI THƯ VIỆN JAVA GIẢI PHƯƠNG TRÌNH BẬC 2

## Mục tiêu
Xây dựng thư viện Java giải phương trình bậc 2 và đóng gói JAR để tái sử dụng.

## Cấu trúc dự án
```text
quadratic-equation-library/
├── pom.xml
├── .gitignore
├── README.md
├── src/main/java/com/codegym/math/QuadraticEquationSolver.java
└── demo/Main.java
```

## File `pom.xml`
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.codegym</groupId>
    <artifactId>quadratic-equation-library</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```

## File `src/main/java/com/codegym/math/QuadraticEquationSolver.java`
```java
package com.codegym.math;

/** Thư viện giải phương trình ax^2 + bx + c = 0 trên tập số thực. */
public final class QuadraticEquationSolver {
    private QuadraticEquationSolver() { }

    public enum Type {
        TWO_ROOTS, DOUBLE_ROOT, ONE_ROOT, NO_REAL_ROOT, INFINITE_SOLUTIONS
    }

    public record Result(Type type, double x1, double x2) { }

    public static Result solve(double a, double b, double c) {
        if (!Double.isFinite(a) || !Double.isFinite(b) || !Double.isFinite(c)) {
            throw new IllegalArgumentException("Các hệ số phải là số hữu hạn.");
        }
        if (a == 0) {
            if (b == 0) {
                return new Result(c == 0 ? Type.INFINITE_SOLUTIONS : Type.NO_REAL_ROOT,
                        Double.NaN, Double.NaN);
            }
            double x = -c / b;
            return new Result(Type.ONE_ROOT, x, x);
        }
        double delta = b * b - 4 * a * c;
        if (delta < 0) return new Result(Type.NO_REAL_ROOT, Double.NaN, Double.NaN);
        if (delta == 0) {
            double x = -b / (2 * a);
            return new Result(Type.DOUBLE_ROOT, x, x);
        }
        double sqrt = Math.sqrt(delta);
        return new Result(Type.TWO_ROOTS, (-b + sqrt) / (2 * a), (-b - sqrt) / (2 * a));
    }
}
```

## File `demo/Main.java`
```java
import com.codegym.math.QuadraticEquationSolver;
import com.codegym.math.QuadraticEquationSolver.Result;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        try (Scanner scanner = new Scanner(System.in)) {
            System.out.println("Giai phuong trinh ax^2 + bx + c = 0");
            System.out.print("Nhap a: "); double a = scanner.nextDouble();
            System.out.print("Nhap b: "); double b = scanner.nextDouble();
            System.out.print("Nhap c: "); double c = scanner.nextDouble();
            Result r = QuadraticEquationSolver.solve(a, b, c);
            switch (r.type()) {
                case TWO_ROOTS -> System.out.printf("Hai nghiem: x1=%.6f; x2=%.6f%n", r.x1(), r.x2());
                case DOUBLE_ROOT -> System.out.printf("Nghiem kep: x=%.6f%n", r.x1());
                case ONE_ROOT -> System.out.printf("Phuong trinh bac nhat: x=%.6f%n", r.x1());
                case NO_REAL_ROOT -> System.out.println("Vo nghiem tren tap so thuc");
                case INFINITE_SOLUTIONS -> System.out.println("Vo so nghiem");
            }
        }
    }
}
```

## File `.gitignore`
```bash
target/
*.class
.idea/
.vscode/
```

## File `README.md`
```markdown
# Thư viện giải phương trình bậc 2 bằng Java

Thư viện Java `com.codegym.math.QuadraticEquationSolver` giải `ax² + bx + c = 0`, xử lý đầy đủ hai nghiệm phân biệt, nghiệm kép, vô nghiệm thực, vô số nghiệm và trường hợp `a = 0` (phương trình bậc nhất).

## Yêu cầu

- JDK 17 trở lên
- Maven 3.8+ (hoặc tương thích)

## Đóng gói thành JAR

Trong thư mục gốc chạy:

```bash
mvn clean package
```

File thư viện được tạo tại `target/quadratic-equation-library-1.0.0.jar`.

## Chạy chương trình dùng thư viện

**Windows (PowerShell):**

```powershell
javac -cp target/quadratic-equation-library-1.0.0.jar -d demo demo/Main.java
java -cp "demo;target/quadratic-equation-library-1.0.0.jar" Main
```

**macOS/Linux:**

```bash
javac -cp target/quadratic-equation-library-1.0.0.jar -d demo demo/Main.java
java -cp "demo:target/quadratic-equation-library-1.0.0.jar" Main
```

Nhập thử `a = 1`, `b = -3`, `c = 2` → nghiệm `x1 = 2`, `x2 = 1`.

## Đưa lên GitHub

Tạo repository trống tên `quadratic-equation-library`, sau đó chạy:

```bash
git init
git add .
git commit -m "feat: create quadratic equation solver Java library"
git branch -M main
git remote add origin https://github.com/USERNAME/quadratic-equation-library.git
git push -u origin main
```

Thay `USERNAME` bằng tài khoản GitHub của bạn. Nộp URL repository; không bắt buộc commit file `.jar` vì có thể tự build bằng Maven.
```
