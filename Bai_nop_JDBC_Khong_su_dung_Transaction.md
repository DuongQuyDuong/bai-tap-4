# BÀI NỘP: JDBC không sử dụng Transaction

**Mục tiêu:** chứng minh `autoCommit=true` làm 2 INSERT được lưu dù UPDATE sau đó lỗi.

**Lưu ý:** Ứng dụng tự xóa và tạo lại bảng `Employee` khi kiểm tra, chỉ dùng trên database thực hành.

## Cấu trúc dự án
```text
user-management-without-transaction/
├── pom.xml
├── .gitignore
├── sql/schema.sql
├── src/main/java/com/codegym/dao/IUserDAO.java
├── src/main/java/com/codegym/dao/UserDAO.java
├── src/main/java/com/codegym/controller/UserServlet.java
├── src/main/webapp/index.jsp
├── src/main/webapp/WEB-INF/web.xml
└── README.md
```


## `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.codegym</groupId>
    <artifactId>user-management</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    <dependencies>
        <dependency>
            <groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version><scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><version>8.4.0</version>
        </dependency>
    </dependencies>
    <build>
        <finalName>user-management</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId><artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```


## `sql/schema.sql`

```sql
CREATE DATABASE IF NOT EXISTS demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE demo;
-- CHỈ dành cho dữ liệu thực hành. Java sẽ tự DROP/CREATE bảng Employee mỗi lần chạy test.
CREATE TABLE IF NOT EXISTS Employee (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(120) NOT NULL,
    salary DECIMAL(15,2) NOT NULL,
    created_Date DATETIME
) ENGINE=InnoDB;
-- Sau khi chạy app:
-- SELECT * FROM Employee;
```


## `src/main/java/com/codegym/dao/IUserDAO.java`

```java
package com.codegym.dao;

import java.sql.SQLException;

public interface IUserDAO {
    void insertUpdateWithoutTransaction() throws SQLException;
}
```


## `src/main/java/com/codegym/dao/UserDAO.java`

```java
package com.codegym.dao;

import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.sql.Statement;
import java.time.LocalDateTime;

public class UserDAO implements IUserDAO {
    private static final String INSERT_SQL =
            "INSERT INTO Employee (name, salary, created_Date) VALUES (?, ?, ?)";
    private static final String UPDATE_SQL =
            "UPDATE Employee SET salary = ? WHERE name = ?";

    private Connection getConnection() throws SQLException {
        String url = System.getenv().getOrDefault(
                "DB_URL", "jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=UTF-8");
        String user = System.getenv().getOrDefault("DB_USER", "root");
        String password = System.getenv().getOrDefault("DB_PASSWORD", "");
        return DriverManager.getConnection(url, user, password);
    }

    @Override
    public void insertUpdateWithoutTransaction() throws SQLException {
        try (Connection conn = getConnection();
             Statement statement = conn.createStatement()) {
            // Chỉ sử dụng ở môi trường thực hành: xoá bảng để mỗi lần test bắt đầu sạch.
            statement.execute("DROP TABLE IF EXISTS Employee");
            statement.execute("CREATE TABLE Employee ("
                    + "id INT AUTO_INCREMENT PRIMARY KEY, "
                    + "name VARCHAR(120) NOT NULL, "
                    + "salary DECIMAL(15,2) NOT NULL, "
                    + "created_Date DATETIME)");

            // Mặc định auto-commit = true, nhưng đặt rõ để minh hoạ.
            conn.setAutoCommit(true);
            try (PreparedStatement psInsert = conn.prepareStatement(INSERT_SQL);
                 PreparedStatement psUpdate = conn.prepareStatement(UPDATE_SQL)) {
                insertEmployee(psInsert, "Quynh", new BigDecimal("10.00"));
                insertEmployee(psInsert, "Ngan", new BigDecimal("20.00"));

                // Cố tình bỏ trống tham số 1: câu UPDATE sẽ thất bại.
                // Đúng phải là psUpdate.setBigDecimal(1, new BigDecimal("999.99"));
                psUpdate.setBigDecimal(2, new BigDecimal("999.99"));
                psUpdate.setString(2, "Quynh");
                psUpdate.executeUpdate();
            }
        }
    }

    private void insertEmployee(PreparedStatement ps, String name, BigDecimal salary)
            throws SQLException {
        ps.setString(1, name);
        ps.setBigDecimal(2, salary);
        ps.setTimestamp(3, Timestamp.valueOf(LocalDateTime.now()));
        ps.executeUpdate();
    }
}
```


## `src/main/java/com/codegym/controller/UserServlet.java`

```java
package com.codegym.controller;

import com.codegym.dao.IUserDAO;
import com.codegym.dao.UserDAO;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
import java.sql.SQLException;

@WebServlet("/users")
public class UserServlet extends HttpServlet {
    private final IUserDAO userDAO = new UserDAO();

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        String action = request.getParameter("action");
        if ("test-without-tran".equals(action)) {
            testWithoutTran(request, response);
        } else {
            response.sendRedirect(request.getContextPath() + "/index.jsp");
        }
    }

    private void testWithoutTran(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        response.setContentType("text/html;charset=UTF-8");
        try (PrintWriter out = response.getWriter()) {
            out.println("<!DOCTYPE html><html lang='vi'><head><meta charset='UTF-8'>"
                    + "<title>JDBC không dùng Transaction</title></head><body>");
            out.println("<h2>Kiểm tra JDBC không sử dụng Transaction</h2>");
            try {
                userDAO.insertUpdateWithoutTransaction();
                out.println("<p>Chạy hoàn tất (không có lỗi).</p>");
            } catch (SQLException e) {
                getServletContext().log("SQL thất bại như kịch bản thực hành", e);
                out.println("<p>UPDATE bị lỗi như dự kiến. Hai INSERT đã được commit.</p>");
                out.println("<p>Kiểm tra MySQL bằng: <code>SELECT * FROM Employee;</code></p>");
            }
            out.println("<p><a href='" + request.getContextPath()
                    + "/index.jsp'>Quay lại</a></p></body></html>");
        }
    }
}
```


## `src/main/webapp/index.jsp`

```xml
<%@ page contentType="text/html;charset=UTF-8" pageEncoding="UTF-8" %>
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Thực hành JDBC không dùng Transaction</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 720px; margin: 60px auto; padding: 20px; }
        a.button { display: inline-block; color: #fff; background: #2356a8; padding: 12px 18px; border-radius: 6px; text-decoration: none; }
        p { line-height: 1.6; }
    </style>
</head>
<body>
    <h1>JDBC không sử dụng Transaction</h1>
    <p>Chạy 2 câu lệnh INSERT và 1 câu lệnh UPDATE cố tình bị lỗi để quan sát dữ liệu ghi một phần.</p>
    <p><strong>Chú ý:</strong> thao tác sẽ xoá và tạo lại bảng Employee trong database demo. Chỉ chạy với dữ liệu thực hành.</p>
    <a class="button" href="${pageContext.request.contextPath}/users?action=test-without-tran">Chạy thử nghiệm</a>
</body>
</html>
```


## `src/main/webapp/WEB-INF/web.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
    <display-name>User Management - Without Transaction</display-name>
    <!-- Servlet được khai báo bằng @WebServlet("/users"), không khai báo trùng ở đây. -->
</web-app>
```


## `.gitignore`

```text
target/
.idea/
.vscode/
*.class
*.log
```


## `README.md`

```markdown
# Bài thực hành: JDBC không sử dụng Transaction

Ứng dụng Java Web sử dụng JSP, Servlet, JDBC, MySQL; Tomcat 10.1+; Java 17+.

## Mục tiêu

Chạy hai INSERT thành công và một UPDATE cố tình lỗi, quan sát **ghi dữ liệu một phần** do `autoCommit=true`.

## Cài đặt và chạy

1. Cài Java 17+, Maven, MySQL và Tomcat 10.1+.
2. Chạy `sql/schema.sql` trên MySQL để tạo database `demo`.
3. Đặt biến môi trường `DB_USER`, `DB_PASSWORD` (và `DB_URL` nếu cần). Mặc định kết nối `jdbc:mysql://localhost:3306/demo` với `root` và mật khẩu rỗng.
4. Tại thư mục dự án chạy `mvn clean package`.
5. Copy `target/user-management.war` vào thư mục `webapps` của Tomcat, khởi động Tomcat.
6. Truy cập `http://localhost:8080/user-management/` và nhấn **Chạy thử nghiệm**, hoặc mở trực tiếp `http://localhost:8080/user-management/users?action=test-without-tran`.
7. Trong MySQL chạy `USE demo; SELECT * FROM Employee;` để xem kết quả.

## Dự kiến

Câu UPDATE thiếu tham số 1 nên báo `SQLException`, nhưng bảng vẫn có 2 bản ghi `Quynh` (10.00) và `Ngan` (20.00). Đó là tác hại khi dùng `autoCommit=true` mà không nhóm các câu lệnh vào transaction.

**Cảnh báo:** Code thực hành có `DROP TABLE IF EXISTS Employee`, sẽ xóa dữ liệu cũ của bảng trong database `demo` mỗi lần bấm chạy. Không sử dụng trên cơ sở dữ liệu thật.

## Nộp GitHub

Tạo repository trống trên GitHub, rồi trong thư mục dự án chạy:

```bash
git init
git add .
git commit -m "feat: demonstrate JDBC without transaction"
git branch -M main
git remote add origin https://github.com/USERNAME/user-management-without-transaction.git
git push -u origin main
```

Thay USERNAME bằng tài khoản GitHub và nộp đường dẫn repository. Không đưa mật khẩu DB lên GitHub.
```
