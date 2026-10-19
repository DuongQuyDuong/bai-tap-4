# Bài thực hành: JDBC Transaction — commit() và rollback()

**Mục tiêu:** Cập nhật ứng dụng quản lý User dùng JDBC transaction. Khi một lệnh SQL bị lỗi trong cùng giao dịch, `rollback()` sẽ hoàn tác toàn bộ thay đổi chưa commit; nếu cả hai lệnh thành công thì gọi `commit()`.

> **Đây là file gộp để nộp/xem mã nguồn, không chạy trực tiếp được.** Muốn triển khai Tomcat 10.1+, hãy tách từng phần mã vào đúng đường dẫn ghi dưới mỗi tiêu đề. Cần Java 17+, Maven, MySQL và Tomcat 10.1+. Đây là ví dụ mở rộng phần thực hành transaction trên ứng dụng User của bài trước.

## Cấu trúc dự án

```text
user-management-transaction/
├── database.sql
├── pom.xml
└── src/main/
    ├── java/com/codegym/model/User.java
    ├── java/com/codegym/dao/UserDAO.java
    ├── java/com/codegym/controller/UserServlet.java
    └── webapp/
        ├── index.jsp
        └── WEB-INF/
            ├── web.xml
            └── views/users.jsp
```

## database.sql

```sql
CREATE DATABASE IF NOT EXISTS demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE demo;
CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(120) NOT NULL,
    email VARCHAR(220) NOT NULL,
    country VARCHAR(120)
);
-- Kiểm tra dữ liệu trước và sau khi thử giao dịch:
SELECT * FROM users ORDER BY id DESC;
```

## pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.codegym</groupId>
    <artifactId>user-management-transaction</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    <dependencies>
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>jakarta.servlet.jsp</groupId>
            <artifactId>jakarta.servlet.jsp-api</artifactId>
            <version>3.1.1</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>jakarta.servlet.jsp.jstl</groupId>
            <artifactId>jakarta.servlet.jsp.jstl-api</artifactId>
            <version>3.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.web</groupId>
            <artifactId>jakarta.servlet.jsp.jstl</artifactId>
            <version>3.0.1</version>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.3.0</version>
        </dependency>
    </dependencies>
    <build>
        <finalName>user-management-transaction</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

## src/main/java/com/codegym/model/User.java

```java
package com.codegym.model;

public class User {
    private int id;
    private String name;
    private String email;
    private String country;

    public User() {}
    public User(String name, String email, String country) {
        this.name = name;
        this.email = email;
        this.country = country;
    }
    public User(int id, String name, String email, String country) {
        this(name, email, country);
        this.id = id;
    }
    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }
}
```

## src/main/java/com/codegym/dao/UserDAO.java

```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class UserDAO {
    // Đổi các biến môi trường DB_URL, DB_USER, DB_PASSWORD theo MySQL của bạn.
    private Connection connect() throws SQLException {
        String url = System.getenv().getOrDefault("DB_URL", "jdbc:mysql://localhost:3306/demo?useSSL=false&allowPublicKeyRetrieval=true&characterEncoding=UTF-8");
        String username = System.getenv().getOrDefault("DB_USER", "root");
        String password = System.getenv().getOrDefault("DB_PASSWORD", "");
        return DriverManager.getConnection(url, username, password);
    }

    public List<User> selectAllUsers() throws SQLException {
        List<User> users = new ArrayList<>();
        try (Connection connection = connect();
             PreparedStatement statement = connection.prepareStatement("SELECT id, name, email, country FROM users ORDER BY id");
             ResultSet rs = statement.executeQuery()) {
            while (rs.next()) {
                users.add(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email"), rs.getString("country")));
            }
        }
        return users;
    }

    /**
     * Thêm 2 user trong cùng một transaction.
     * forceSqlError = true: cố ý chạy SQL sai sau INSERT thứ nhất để quan sát rollback.
     * forceSqlError = false: cả 2 INSERT thành công và commit.
     * LƯU Ý: chỉ sử dụng cách gây lỗi này cho bài thực hành, không dùng trên production.
     */
    public void addUserTransaction(User first, User second, boolean forceSqlError) throws SQLException {
        String insertSql = "INSERT INTO users (name, email, country) VALUES (?, ?, ?)";
        try (Connection connection = connect()) {
            connection.setAutoCommit(false); // Tắt auto-commit
            try {
                try (PreparedStatement ps = connection.prepareStatement(insertSql)) {
                    ps.setString(1, first.getName());
                    ps.setString(2, first.getEmail());
                    ps.setString(3, first.getCountry());
                    ps.executeUpdate(); // INSERT 1: mới chỉ nằm trong transaction
                }

                if (forceSqlError) {
                    // Chủ đích tạo lỗi SQL: cột non_existing_column không tồn tại.
                    try (PreparedStatement broken = connection.prepareStatement(
                            "INSERT INTO users (non_existing_column) VALUES (?)")) {
                        broken.setString(1, second.getName());
                        broken.executeUpdate(); // Lỗi -> nhảy vào catch -> rollback
                    }
                } else {
                    try (PreparedStatement ps = connection.prepareStatement(insertSql)) {
                        ps.setString(1, second.getName());
                        ps.setString(2, second.getEmail());
                        ps.setString(3, second.getCountry());
                        ps.executeUpdate();
                    }
                }

                connection.commit(); // Chỉ commit sau khi mọi câu SQL thành công
            } catch (SQLException ex) {
                try {
                    connection.rollback(); // Hoàn tác INSERT 1 khi INSERT 2 lỗi
                } catch (SQLException rollbackError) {
                    ex.addSuppressed(rollbackError);
                }
                throw ex;
            } finally {
                connection.setAutoCommit(true);
            }
        }
    }
}
```

## src/main/java/com/codegym/controller/UserServlet.java

```java
package com.codegym.controller;

import com.codegym.dao.UserDAO;
import com.codegym.model.User;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.sql.SQLException;

@WebServlet("/users")
public class UserServlet extends HttpServlet {
    private final UserDAO dao = new UserDAO();

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            request.setAttribute("users", dao.selectAllUsers());
            request.getRequestDispatcher("/WEB-INF/views/users.jsp").forward(request, response);
        } catch (SQLException e) {
            throw new ServletException("Không thể đọc dữ liệu MySQL", e);
        }
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        request.setCharacterEncoding("UTF-8");
        User a = new User(request.getParameter("name1"), request.getParameter("email1"), request.getParameter("country1"));
        User b = new User(request.getParameter("name2"), request.getParameter("email2"), request.getParameter("country2"));
        boolean forceError = "on".equals(request.getParameter("forceError"));
        String message;
        try {
            dao.addUserTransaction(a, b, forceError);
            message = "COMMIT thành công: đã lưu cả 2 User.";
        } catch (SQLException e) {
            message = "ROLLBACK: giao dịch lỗi và cả 2 User không được thêm. Chi tiết: " + e.getMessage();
        }
        try {
            request.setAttribute("message", message);
            request.setAttribute("users", dao.selectAllUsers());
            request.getRequestDispatcher("/WEB-INF/views/users.jsp").forward(request, response);
        } catch (SQLException e) {
            throw new ServletException("Không thể đọc dữ liệu MySQL", e);
        }
    }
}
```

## src/main/webapp/WEB-INF/views/users.jsp

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
<!doctype html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>JDBC Transaction - Quản lý User</title>
  <style>
    body { font-family: Arial, sans-serif; background:#f5f7fb; color:#243047; padding:24px; }
    main { max-width:850px; margin:auto; } .card { background:white; padding:24px; margin-bottom:18px; border-radius:12px; box-shadow:0 3px 14px #1b2a7a14; }
    .row { display:grid; grid-template-columns:repeat(auto-fit,minmax(180px,1fr)); gap:12px; }
    label { display:block; font-weight:bold; font-size:14px; } input { width:100%; margin-top:5px; padding:10px; box-sizing:border-box; }
    button { padding:12px 18px; border:0; border-radius:7px; background:#1b2a7a; color:white; cursor:pointer; }
    table { width:100%; border-collapse:collapse; } th,td { padding:10px; border-bottom:1px solid #e5e9f0; text-align:left; }
    .note { color:#6a7282; } .message { padding:12px; background:#ecf3ff; border-radius:6px; }
  </style>
</head>
<body><main>
  <h1>Thực hành JDBC Transaction</h1>
  <div class="card">
    <h2>Thêm 2 User trong một giao dịch</h2>
    <p class="note">Bật tùy chọn gây lỗi để kiểm thử rollback().</p>
    <c:if test="${not empty message}"><p class="message"><c:out value="${message}" /></p></c:if>
    <form action="${pageContext.request.contextPath}/users" method="post">
      <h3>User thứ nhất</h3>
      <div class="row">
        <label>Tên<input name="name1" value="Minh Test" required /></label>
        <label>Email<input type="email" name="email1" value="minh.test@example.com" required /></label>
        <label>Quốc gia<input name="country1" value="Viet Nam" required /></label>
      </div>
      <h3>User thứ hai</h3>
      <div class="row">
        <label>Tên<input name="name2" value="Lan Test" required /></label>
        <label>Email<input type="email" name="email2" value="lan.test@example.com" required /></label>
        <label>Quốc gia<input name="country2" value="Viet Nam" required /></label>
      </div>
      <p><label><input type="checkbox" name="forceError" checked style="width:auto" /> Cố ý gây lỗi SQL (để kiểm tra rollback)</label></p>
      <button type="submit">Thực hiện transaction</button>
    </form>
  </div>
  <div class="card">
    <h2>Danh sách User trong MySQL</h2>
    <table><thead><tr><th>ID</th><th>Tên</th><th>Email</th><th>Quốc gia</th></tr></thead>
      <tbody><c:forEach var="u" items="${users}"><tr>
        <td><c:out value="${u.id}" /></td><td><c:out value="${u.name}" /></td>
        <td><c:out value="${u.email}" /></td><td><c:out value="${u.country}" /></td>
      </tr></c:forEach></tbody>
    </table>
  </div>
</main></body></html>
```

## src/main/webapp/WEB-INF/web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
    <display-name>User Management Transaction</display-name>
</web-app>
```

## src/main/webapp/index.jsp

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<% response.sendRedirect(request.getContextPath() + "/users"); %>
```

## Cách thử commit() và rollback()

1. Chạy `database.sql` trên MySQL để tạo database `demo` và bảng `users` (InnoDB là mặc định trên MySQL thông thường). Đảm bảo database và bảng dùng engine hỗ trợ transaction, ví dụ InnoDB.
2. Đặt biến môi trường `DB_URL`, `DB_USER`, `DB_PASSWORD` nếu cần (mặc định `root` và mật khẩu rỗng). Không đưa mật khẩu thật vào GitHub.
3. Tách mã nguồn ra đúng các file, mở terminal ở thư mục có `pom.xml`, chạy `mvn clean package`, copy `target/user-management-transaction.war` vào thư mục `webapps` của Tomcat 10.1+ và khởi động Tomcat.
4. Truy cập `http://localhost:8080/user-management-transaction/users`.
5. **Kiểm tra rollback:** giữ ô “Cố ý gây lỗi SQL” được chọn rồi gửi. INSERT đầu tiên chạy nhưng INSERT tiếp theo sử dụng cột không tồn tại nên phát sinh `SQLException`, `rollback()` được gọi. Chạy `SELECT * FROM demo.users;`: không có 2 user vừa gửi.
6. **Kiểm tra commit:** bỏ chọn ô gây lỗi, gửi lại. Hai INSERT thành công, `commit()` được gọi và cả hai User xuất hiện trong bảng.

**Lưu ý:** Đây là lỗi SQL được tạo có chủ đích trong môi trường thực hành; đừng áp dụng mã gây lỗi này vào ứng dụng thực tế. Nếu kiểm tra AUTO_INCREMENT thấy ID nhảy số sau rollback, điều đó có thể xảy ra và không chứng minh rollback thất bại. Cần kiểm tra bản ghi thực tế thay vì chuỗi ID. Bài chưa được kiểm thử trực tiếp với MySQL/Tomcat trong môi trường này.
