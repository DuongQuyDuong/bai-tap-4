# BÀI NỘP: STORED PROCEDURES VỚI JDBC - QUẢN LÝ USER

**Mục tiêu:** Gọi Stored Procedures thông qua `CallableStatement` để liệt kê, cập nhật, xóa User.

**Môi trường:** JDK 17, Maven, Tomcat 10.1+, MySQL 8, JSP/JSTL Jakarta.

**Cấu trúc dự án:**

```text
user-management-stored-procedures/
  .gitignore
  README.md
  database.sql
  pom.xml
  src/main/java/com/codegym/dao/IUserDAO.java
  src/main/java/com/codegym/dao/UserDAO.java
  src/main/java/com/codegym/model/User.java
  src/main/java/com/codegym/web/UserServlet.java
  src/main/webapp/WEB-INF/views/edit.jsp
  src/main/webapp/WEB-INF/views/list.jsp
  src/main/webapp/WEB-INF/web.xml
  src/main/webapp/index.jsp
```

> Mỗi mục dưới đây là nội dung của một file. Để chạy hoặc upload GitHub, tạo đúng các file theo đường dẫn; không upload duy nhất file Markdown này.

## `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.codegym</groupId><artifactId>user-management</artifactId><version>1.0-SNAPSHOT</version><packaging>war</packaging>
  <properties><maven.compiler.release>17</maven.compiler.release><project.build.sourceEncoding>UTF-8</project.build.sourceEncoding></properties>
  <dependencies>
    <dependency><groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId><version>6.0.0</version><scope>provided</scope></dependency>
    <dependency><groupId>jakarta.servlet.jsp.jstl</groupId><artifactId>jakarta.servlet.jsp.jstl-api</artifactId><version>3.0.0</version></dependency>
    <dependency><groupId>org.glassfish.web</groupId><artifactId>jakarta.servlet.jsp.jstl</artifactId><version>3.0.1</version></dependency>
    <dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><version>8.4.0</version></dependency>
  </dependencies>
  <build><finalName>user-management</finalName><plugins><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId><version>3.13.0</version></plugin><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-war-plugin</artifactId><version>3.4.0</version></plugin></plugins></build>
</project>
```

## `src/main/java/com/codegym/model/User.java`

```java
package com.codegym.model;

public class User {
    private int id;
    private String name;
    private String email;
    private String country;

    public User(int id, String name, String email, String country) {
        this.id = id; this.name = name; this.email = email; this.country = country;
    }
    public int getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getCountry() { return country; }
}
```

## `src/main/java/com/codegym/dao/IUserDAO.java`

```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.SQLException;
import java.util.List;

public interface IUserDAO {
    List<User> selectAllUsers() throws SQLException;
    User selectUser(int id) throws SQLException;
    boolean updateUser(User user) throws SQLException;
    boolean deleteUser(int id) throws SQLException;
}
```

## `src/main/java/com/codegym/dao/UserDAO.java`

```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class UserDAO implements IUserDAO {
    private static final String URL = "jdbc:mysql://localhost:3306/user_management?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Bangkok";
    private static final String USERNAME = "root";
    private static final String PASSWORD = "CHANGE_ME"; // Thay bằng mật khẩu MySQL của bạn

    private Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USERNAME, PASSWORD);
    }

    @Override
    public List<User> selectAllUsers() throws SQLException {
        List<User> users = new ArrayList<>();
        try (Connection conn = getConnection();
             CallableStatement call = conn.prepareCall("{CALL sp_get_all_users()}");
             ResultSet rs = call.executeQuery()) {
            while (rs.next()) {
                users.add(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email"), rs.getString("country")));
            }
        }
        return users;
    }

    @Override
    public User selectUser(int id) throws SQLException {
        String sql = "SELECT id, name, email, country FROM users WHERE id = ?";
        try (Connection conn = getConnection(); PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) return new User(rs.getInt("id"), rs.getString("name"), rs.getString("email"), rs.getString("country"));
            }
        }
        return null;
    }

    @Override
    public boolean updateUser(User user) throws SQLException {
        try (Connection conn = getConnection();
             CallableStatement call = conn.prepareCall("{CALL sp_update_user(?, ?, ?, ?)}")) {
            call.setInt(1, user.getId());
            call.setString(2, user.getName());
            call.setString(3, user.getEmail());
            call.setString(4, user.getCountry());
            return call.executeUpdate() > 0;
        }
    }

    @Override
    public boolean deleteUser(int id) throws SQLException {
        try (Connection conn = getConnection();
             CallableStatement call = conn.prepareCall("{CALL sp_delete_user(?)}")) {
            call.setInt(1, id);
            return call.executeUpdate() > 0;
        }
    }
}
```

## `src/main/java/com/codegym/web/UserServlet.java`

```java
package com.codegym.web;

import com.codegym.dao.IUserDAO;
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
    private final IUserDAO dao = new UserDAO();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String action = req.getParameter("action");
        if ("edit".equals(action)) {
            try {
                int id = Integer.parseInt(req.getParameter("id"));
                User user = dao.selectUser(id);
                if (user == null) { resp.sendError(404, "Không tìm thấy user"); return; }
                req.setAttribute("user", user);
                req.getRequestDispatcher("/WEB-INF/views/edit.jsp").forward(req, resp);
            } catch (NumberFormatException e) { resp.sendError(400, "ID không hợp lệ"); }
              catch (SQLException e) { throw new ServletException(e); }
        } else {
            try {
                req.setAttribute("users", dao.selectAllUsers());
                req.getRequestDispatcher("/WEB-INF/views/list.jsp").forward(req, resp);
            } catch (SQLException e) { throw new ServletException(e); }
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        req.setCharacterEncoding("UTF-8");
        String action = req.getParameter("action");
        try {
            int id = Integer.parseInt(req.getParameter("id"));
            if ("update".equals(action)) {
                String name = req.getParameter("name");
                String email = req.getParameter("email");
                String country = req.getParameter("country");
                if (name == null || name.isBlank() || email == null || email.isBlank() || country == null || country.isBlank()) {
                    resp.sendError(400, "Thông tin không được để trống"); return;
                }
                dao.updateUser(new User(id, name.trim(), email.trim(), country.trim()));
            } else if ("delete".equals(action)) {
                dao.deleteUser(id);
            } else { resp.sendError(400, "Action không hợp lệ"); return; }
            resp.sendRedirect(req.getContextPath() + "/users");
        } catch (NumberFormatException e) { resp.sendError(400, "ID không hợp lệ"); }
          catch (SQLException e) { throw new ServletException(e); }
    }
}
```

## `src/main/webapp/index.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<% response.sendRedirect(request.getContextPath() + "/users"); %>
```

## `src/main/webapp/WEB-INF/views/list.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
<!DOCTYPE html>
<html lang="vi"><head><meta charset="UTF-8"><title>Danh sách User</title>
<style>body{font-family:Arial;margin:40px}table{border-collapse:collapse;width:100%}td,th{padding:12px;border:1px solid #ddd}th{background:#e9efff}button{cursor:pointer}</style></head>
<body><h1>Danh sách User (Stored Procedures)</h1>
<table><thead><tr><th>ID</th><th>Tên</th><th>Email</th><th>Quốc gia</th><th>Thao tác</th></tr></thead>
<tbody><c:forEach var="u" items="${users}"><tr>
<td><c:out value="${u.id}"/></td><td><c:out value="${u.name}"/></td><td><c:out value="${u.email}"/></td><td><c:out value="${u.country}"/></td>
<td><c:url var="editUrl" value="/users"><c:param name="action" value="edit"/><c:param name="id" value="${u.id}"/></c:url>
<a href="${editUrl}">Sửa</a>
<form method="post" action="${pageContext.request.contextPath}/users" style="display:inline" onsubmit="return confirm('Xóa user này?')">
<input type="hidden" name="action" value="delete"><input type="hidden" name="id" value="${u.id}"><button type="submit">Xóa</button></form></td>
</tr></c:forEach></tbody></table></body></html>
```

## `src/main/webapp/WEB-INF/views/edit.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
<!DOCTYPE html><html lang="vi"><head><meta charset="UTF-8"><title>Sửa user</title>
<style>body{font-family:Arial;margin:40px}label{display:block;margin:12px 0}input{padding:7px;width:340px}</style></head>
<body><h1>Cập nhật User #<c:out value="${user.id}"/></h1>
<form method="post" action="${pageContext.request.contextPath}/users">
<input type="hidden" name="action" value="update"><input type="hidden" name="id" value="${user.id}">
<label>Tên <input name="name" value="${user.name}" required></label>
<label>Email <input type="email" name="email" value="${user.email}" required></label>
<label>Country <input name="country" value="${user.country}" required></label>
<button type="submit">Lưu</button> <a href="${pageContext.request.contextPath}/users">Quay lại</a>
</form></body></html>
```

## `src/main/webapp/WEB-INF/web.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd" version="6.0">
  <display-name>User Management Stored Procedures</display-name>
</web-app>
```

## `database.sql`

```sql
CREATE DATABASE IF NOT EXISTS user_management CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE user_management;

CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL,
    country VARCHAR(100) NOT NULL
) ENGINE=InnoDB;

INSERT INTO users (name, email, country) VALUES
('Nguyen Van A', 'a@example.com', 'Vietnam'),
('Tran Thi B', 'b@example.com', 'Japan'),
('Le Van C', 'c@example.com', 'Vietnam');

DROP PROCEDURE IF EXISTS sp_get_all_users;
DROP PROCEDURE IF EXISTS sp_update_user;
DROP PROCEDURE IF EXISTS sp_delete_user;
DELIMITER $$
CREATE PROCEDURE sp_get_all_users()
BEGIN
    SELECT id, name, email, country FROM users ORDER BY id;
END$$

CREATE PROCEDURE sp_update_user(
    IN p_id INT, IN p_name VARCHAR(100), IN p_email VARCHAR(150), IN p_country VARCHAR(100)
)
BEGIN
    UPDATE users SET name = p_name, email = p_email, country = p_country WHERE id = p_id;
END$$

CREATE PROCEDURE sp_delete_user(IN p_id INT)
BEGIN
    DELETE FROM users WHERE id = p_id;
END$$
DELIMITER ;

-- Thử riêng trong MySQL:
-- CALL sp_get_all_users();
-- CALL sp_update_user(1, 'Nguyen Van A Updated', 'new@example.com', 'Vietnam');
-- CALL sp_delete_user(2);
```

## `.gitignore`

```text
target/
.idea/
.vscode/
*.class
```

## `README.md`

```markdown
# Quản lý User - Stored Procedures JDBC

Bài thực hành Java Web MVC với JSP, Servlet, JDBC, MySQL và `CallableStatement`.

## Chức năng
- Hiển thị danh sách User thông qua `sp_get_all_users()`.
- Sửa User thông qua `sp_update_user(...)`.
- Xóa User thông qua `sp_delete_user(...)`.
- Chọn user để sửa bằng `PreparedStatement`.

## Chạy
1. Cài JDK 17+, Maven, MySQL 8+, Tomcat 10.1+.
2. Chạy `database.sql` trong MySQL (nếu đang có dữ liệu, chỉ tạo procedures thay vì chạy toàn bộ file để tránh thêm dữ liệu mẫu trùng).
3. Sửa `USERNAME`, `PASSWORD`, `URL` trong `UserDAO.java`.
4. Chạy `mvn clean package`.
5. Copy `target/user-management.war` vào thư mục `webapps` của Tomcat rồi khởi động Tomcat.
6. Truy cập http://localhost:8080/user-management/users

## Kiểm thử
- Danh sách: kiểm tra 3 bản ghi ban đầu.
- Sửa: đổi tên/country/email, lưu rồi kiểm tra trên giao diện và MySQL.
- Xóa: bấm Xóa và xác nhận, kiểm tra bản ghi không còn.

## Nộp GitHub
Tạo repository trống `user-management-stored-procedures`, rồi chạy:
```bash
git init
git add .
git commit -m "feat: user management via jdbc stored procedures"
git branch -M main
git remote add origin https://github.com/USERNAME/user-management-stored-procedures.git
git push -u origin main
```
Thay USERNAME bằng username GitHub của bạn. Không đưa mật khẩu thật lên GitHub (đổi mật khẩu mẫu trước khi commit hoặc dùng biến môi trường).
```

