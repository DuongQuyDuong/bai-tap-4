# BÀI NỘP — JDBC TRANSACTION (COMMIT / ROLLBACK)

Dự án: `user-jdbc-transaction` — Java 17, Tomcat 10.1+, MySQL.

## Cấu trúc thư mục
```text
user-jdbc-transaction/.gitignore
user-jdbc-transaction/README.md
user-jdbc-transaction/pom.xml
user-jdbc-transaction/sql/schema.sql
user-jdbc-transaction/src/main/java/com/codegym/dao/IUserDAO.java
user-jdbc-transaction/src/main/java/com/codegym/dao/UserDAO.java
user-jdbc-transaction/src/main/java/com/codegym/model/User.java
user-jdbc-transaction/src/main/java/com/codegym/web/UserServlet.java
user-jdbc-transaction/src/main/webapp/WEB-INF/views/users.jsp
user-jdbc-transaction/src/main/webapp/WEB-INF/web.xml
user-jdbc-transaction/src/main/webapp/index.jsp
```

## Toàn bộ mã nguồn

### `pom.xml`
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion><groupId>com.codegym</groupId><artifactId>user-jdbc-transaction</artifactId><version>1.0</version><packaging>war</packaging>
  <properties><maven.compiler.release>17</maven.compiler.release><project.build.sourceEncoding>UTF-8</project.build.sourceEncoding></properties>
  <dependencies>
    <dependency><groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId><version>6.0.0</version><scope>provided</scope></dependency>
    <dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><version>8.4.0</version></dependency>
  </dependencies>
  <build><finalName>user-jdbc-transaction</finalName><plugins><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId><version>3.13.0</version></plugin><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-war-plugin</artifactId><version>3.4.0</version></plugin></plugins></build>
</project>
```

### `sql/schema.sql`
```sql
CREATE DATABASE IF NOT EXISTS user_management CHARACTER SET utf8mb4;
USE user_management;
CREATE TABLE IF NOT EXISTS users (
 id INT AUTO_INCREMENT PRIMARY KEY,
 name VARCHAR(100) NOT NULL,
 email VARCHAR(120) NOT NULL UNIQUE,
 country VARCHAR(100) NOT NULL
) ENGINE=InnoDB;
-- Dữ liệu để đối chiếu trước và sau transaction:
SELECT * FROM users ORDER BY id;
```

### `src/main/java/com/codegym/model/User.java`
```java
package com.codegym.model;

public class User {
    private final int id;
    private final String name, email, country;
    public User(int id, String name, String email, String country) {
        this.id = id; this.name = name; this.email = email; this.country = country;
    }
    public int getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getCountry() { return country; }
}
```

### `src/main/java/com/codegym/dao/IUserDAO.java`
```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.SQLException;
import java.util.List;

public interface IUserDAO {
    List<User> findAll() throws SQLException;
    void addUserTransaction(boolean simulateError) throws SQLException;
}
```

### `src/main/java/com/codegym/dao/UserDAO.java`
```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class UserDAO implements IUserDAO {
    private static final String URL = System.getenv().getOrDefault("DB_URL", "jdbc:mysql://localhost:3306/user_management?useSSL=false&serverTimezone=Asia/Bangkok");
    private static final String USER = System.getenv().getOrDefault("DB_USER", "root");
    private static final String PASSWORD = System.getenv().getOrDefault("DB_PASSWORD", "");
    private Connection getConnection() throws SQLException { return DriverManager.getConnection(URL, USER, PASSWORD); }

    @Override public List<User> findAll() throws SQLException {
        List<User> result = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement("SELECT id, name, email, country FROM users ORDER BY id");
             ResultSet rs = ps.executeQuery()) {
            while (rs.next()) result.add(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email"), rs.getString("country")));
        }
        return result;
    }

    @Override public void addUserTransaction(boolean simulateError) throws SQLException {
        String insert = "INSERT INTO users(name, email, country) VALUES (?, ?, ?)";
        try (Connection conn = getConnection()) {
            boolean previousAutoCommit = conn.getAutoCommit();
            try {
                conn.setAutoCommit(false);
                try (PreparedStatement ps = conn.prepareStatement(insert)) {
                    ps.setString(1, "Nguyen Van A");
                    ps.setString(2, "tran_a_" + System.nanoTime() + "@example.com");
                    ps.setString(3, "Vietnam");
                    ps.executeUpdate();

                    ps.setString(1, "Tran Thi B");
                    ps.setString(2, "tran_b_" + System.nanoTime() + "@example.com");
                    ps.setString(3, "Vietnam");
                    ps.executeUpdate();
                }
                if (simulateError) {
                    // Cố ý dùng cột không tồn tại: tạo SQLException để quan sát rollback.
                    try (Statement broken = conn.createStatement()) {
                        broken.executeUpdate("UPDATE users SET invalid_column = 1 WHERE 1 = 0");
                    }
                }
                conn.commit();
            } catch (SQLException | RuntimeException e) {
                try { conn.rollback(); } catch (SQLException rollbackError) { e.addSuppressed(rollbackError); }
                throw e;
            } finally {
                try { conn.setAutoCommit(previousAutoCommit); } catch (SQLException ignored) { /* Connection sẽ được đóng */ }
            }
        }
    }
}
```

### `src/main/java/com/codegym/web/UserServlet.java`
```java
package com.codegym.web;

import com.codegym.dao.IUserDAO;
import com.codegym.dao.UserDAO;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.sql.SQLException;

@WebServlet("/users")
public class UserServlet extends HttpServlet {
    private final IUserDAO dao = new UserDAO();

    @Override protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            req.setAttribute("users", dao.findAll());
            req.getRequestDispatcher("/WEB-INF/views/users.jsp").forward(req, resp);
        } catch (SQLException e) { throw new ServletException("Không tải được danh sách user", e); }
    }

    @Override protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
        boolean error = "rollback".equals(req.getParameter("mode"));
        try {
            dao.addUserTransaction(error);
            req.getSession().setAttribute("message", "COMMIT thành công: đã thêm cả 2 user.");
        } catch (SQLException e) {
            req.getSession().setAttribute("message", "SQL lỗi → đã gọi ROLLBACK: " + e.getMessage());
        }
        resp.sendRedirect(req.getContextPath() + "/users");
    }
}
```

### `src/main/webapp/index.jsp`
```xml
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<% response.sendRedirect(request.getContextPath() + "/users"); %>
```

### `src/main/webapp/WEB-INF/views/users.jsp`
```xml
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="java.util.List,com.codegym.model.User" %>
<!DOCTYPE html><html lang="vi"><head><meta charset="UTF-8"><title>JDBC Transaction</title>
<style>body{font-family:Arial,sans-serif;max-width:850px;margin:40px auto;padding:0 15px}table{width:100%;border-collapse:collapse}th,td{border:1px solid #ccc;padding:9px;text-align:left}button{padding:10px 15px;cursor:pointer;margin-right:10px}</style></head><body>
<h1>Quản lý User - JDBC Transaction</h1>
<p>Thử nghiệm 2 INSERT trong cùng một transaction: khi lỗi sẽ rollback cả hai.</p>
<% String message = (String)session.getAttribute("message"); if (message != null) { %>
<p><strong><%= message.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;").replace(""", "&quot;").replace("'", "&#39;") %></strong></p>
<% session.removeAttribute("message"); } %>
<form method="post" action="<%= request.getContextPath() %>/users">
<button type="submit" name="mode" value="commit">Thử COMMIT</button>
<button type="submit" name="mode" value="rollback">Thử ROLLBACK (cố tình lỗi SQL)</button>
</form><h2>Danh sách người dùng</h2>
<table><thead><tr><th>ID</th><th>Họ tên</th><th>Email</th><th>Quốc gia</th></tr></thead><tbody>
<% for(User user : (List<User>)request.getAttribute("users")) { %>
<tr><td><%= user.getId() %></td><td><%= user.getName() %></td><td><%= user.getEmail() %></td><td><%= user.getCountry() %></td></tr>
<% } %>
</tbody></table></body></html>
```

### `src/main/webapp/WEB-INF/web.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="6.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd">
<display-name>User JDBC Transaction</display-name></web-app>
```

### `.gitignore`
```text
target/
.idea/
.vscode/
*.class
```

### `README.md`
```text
# Bài thực hành JDBC Transaction — Quản lý User

Ứng dụng Maven WAR sử dụng Java 17, JSP, Servlet Jakarta (Tomcat 10.1+), MySQL 8 và JDBC.

## Chạy thử

1. Chạy script `sql/schema.sql` trên MySQL (bảng dùng InnoDB).
2. Cấu hình biến môi trường `DB_URL`, `DB_USER`, `DB_PASSWORD`; mặc định kết nối `localhost:3306/user_management` bằng root và mật khẩu rỗng (chỉ phù hợp máy học tập).
3. Chạy `mvn clean package`.
4. Copy `target/user-jdbc-transaction.war` vào thư mục `webapps` của Tomcat 10.1+ và khởi động server.
5. Truy cập `http://localhost:8080/user-jdbc-transaction/users`.
6. Nhấn **Thử ROLLBACK**: 2 INSERT sẽ chạy rồi UPDATE cột không tồn tại gây lỗi; `rollback()` sẽ hủy 2 INSERT.
7. Nhấn **Thử COMMIT**: 2 INSERT thành công và được ghi lại.
8. Kiểm tra bằng `SELECT * FROM user_management.users ORDER BY id;` trước và sau từng thao tác.

**Lưu ý:** `AUTO_INCREMENT` có thể tăng ngay cả sau rollback, nhưng không có 2 bản ghi mới được lưu. Việc `DROP TABLE` hoặc `CREATE TABLE` bên trong transaction trên MySQL có thể gây implicit commit, nên bài này tạo bảng bằng script riêng. Ví dụ dùng `rollback()` tường minh, không phụ thuộc vào đóng Connection.

## Đưa lên GitHub

Tạo repository mới `user-jdbc-transaction`, sau đó chạy trong thư mục dự án:

```bash
git init
git add .
git commit -m "feat: practice JDBC commit and rollback"
git branch -M main
git remote add origin https://github.com/USERNAME/user-jdbc-transaction.git
git push -u origin main
```

Thay `USERNAME` bằng tên tài khoản GitHub. Nộp link repository, không nộp link file Markdown gộp.
```
