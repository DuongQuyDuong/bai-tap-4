# BÀI NỘP: JDBC TRANSACTION – THÊM USER VÀ PHÂN QUYỀN

## Mục tiêu và nguyên lý
Sử dụng `Connection.setAutoCommit(false)`, `commit()` và `rollback()` để đảm bảo việc thêm User và gán Permissions thành công hoặc thất bại cùng nhau. Dữ liệu lưu trong MySQL (InnoDB).

## Cấu trúc mã nguồn
```text
user-permission-transaction/
  .gitignore
  README.md
  pom.xml
  sql/demo.sql
  src/main/java/com/codegym/controller/UserServlet.java
  src/main/java/com/codegym/dao/IUserDAO.java
  src/main/java/com/codegym/dao/UserDAO.java
  src/main/java/com/codegym/model/User.java
  src/main/webapp/WEB-INF/web.xml
  src/main/webapp/index.jsp
  src/main/webapp/user/create.jsp
  src/main/webapp/user/list.jsp
```

## Toàn bộ nội dung file

### `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.codegym</groupId><artifactId>user-management</artifactId><version>1.0-SNAPSHOT</version><packaging>war</packaging>
  <properties><maven.compiler.release>17</maven.compiler.release><project.build.sourceEncoding>UTF-8</project.build.sourceEncoding></properties>
  <dependencies>
    <dependency><groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId><version>6.0.0</version><scope>provided</scope></dependency>
    <dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><version>8.4.0</version></dependency>
  </dependencies>
  <build><finalName>user-management</finalName><plugins><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId><version>3.13.0</version></plugin><plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-war-plugin</artifactId><version>3.4.0</version></plugin></plugins></build>
</project>
```

### `sql/demo.sql`

```sql
CREATE DATABASE IF NOT EXISTS demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE demo;
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(120) NOT NULL,
  country VARCHAR(120) NOT NULL
) ENGINE=InnoDB;
CREATE TABLE IF NOT EXISTS permission (
  id INT PRIMARY KEY,
  name VARCHAR(50) NOT NULL
) ENGINE=InnoDB;
CREATE TABLE IF NOT EXISTS user_permission (
  user_id INT NOT NULL,
  permission_id INT NOT NULL,
  PRIMARY KEY(user_id, permission_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (permission_id) REFERENCES permission(id)
) ENGINE=InnoDB;
INSERT INTO permission (id, name) VALUES
  (1,'add'),(2,'edit'),(3,'delete'),(4,'view')
ON DUPLICATE KEY UPDATE name=VALUES(name);
```

### `src/main/java/com/codegym/model/User.java`

```java
package com.codegym.model;

public class User {
    private final int id;
    private final String name;
    private final String email;
    private final String country;

    public User(String name, String email, String country) {
        this(0, name, email, country);
    }
    public User(int id, String name, String email, String country) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.country = country;
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
    void addUserTransaction(User user, int[] permissionIds) throws SQLException;
    List<User> findAll() throws SQLException;
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
    // Thiết lập bằng biến môi trường; không commit mật khẩu thực lên GitHub.
    private static final String URL = System.getenv().getOrDefault("DB_URL", "jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Bangkok");
    private static final String USERNAME = System.getenv().getOrDefault("DB_USER", "root");
    private static final String PASSWORD = System.getenv().getOrDefault("DB_PASSWORD", "");

    private Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USERNAME, PASSWORD);
    }

    @Override
    public void addUserTransaction(User user, int[] permissionIds) throws SQLException {
        String insertUser = "INSERT INTO users (name, email, country) VALUES (?, ?, ?)";
        String insertPermission = "INSERT INTO user_permission (user_id, permission_id) VALUES (?, ?)";
        try (Connection conn = getConnection()) {
            conn.setAutoCommit(false);
            try {
                int userId;
                try (PreparedStatement ps = conn.prepareStatement(insertUser, Statement.RETURN_GENERATED_KEYS)) {
                    ps.setString(1, user.getName());
                    ps.setString(2, user.getEmail());
                    ps.setString(3, user.getCountry());
                    ps.executeUpdate();
                    try (ResultSet keys = ps.getGeneratedKeys()) {
                        if (!keys.next()) throw new SQLException("Không lấy được ID User vừa tạo");
                        userId = keys.getInt(1);
                    }
                }
                if (permissionIds != null) {
                    try (PreparedStatement ps = conn.prepareStatement(insertPermission)) {
                        for (int permissionId : permissionIds) {
                            ps.setInt(1, userId);
                            ps.setInt(2, permissionId);
                            ps.executeUpdate();
                        }
                    }
                }
                conn.commit();
            } catch (SQLException | RuntimeException e) {
                try { conn.rollback(); }
                catch (SQLException rollbackError) { e.addSuppressed(rollbackError); }
                throw e;
            } finally {
                conn.setAutoCommit(true);
            }
        }
    }

    @Override
    public List<User> findAll() throws SQLException {
        List<User> users = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement("SELECT id, name, email, country FROM users ORDER BY id DESC");
             ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                users.add(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email"), rs.getString("country")));
            }
        }
        return users;
    }
}
```

### `src/main/java/com/codegym/controller/UserServlet.java`

```java
package com.codegym.controller;

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
    private final IUserDAO userDAO = new UserDAO();

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String action = request.getParameter("action");
        if ("create".equals(action)) {
            request.getRequestDispatcher("/user/create.jsp").forward(request, response);
            return;
        }
        try {
            request.setAttribute("users", userDAO.findAll());
            request.getRequestDispatcher("/user/list.jsp").forward(request, response);
        } catch (SQLException e) {
            throw new ServletException("Không tải được danh sách User", e);
        }
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.setCharacterEncoding("UTF-8");
        String name = request.getParameter("name");
        String email = request.getParameter("email");
        String country = request.getParameter("country");
        if (name == null || name.isBlank() || email == null || email.isBlank() || country == null || country.isBlank()) {
            request.setAttribute("error", "Vui lòng nhập đầy đủ thông tin");
            request.getRequestDispatcher("/user/create.jsp").forward(request, response);
            return;
        }
        String[] selected = request.getParameterValues("permissions");
        int[] permissionIds = new int[selected == null ? 0 : selected.length];
        try {
            for (int i = 0; i < permissionIds.length; i++) permissionIds[i] = Integer.parseInt(selected[i]);
            userDAO.addUserTransaction(new User(name.trim(), email.trim(), country.trim()), permissionIds);
            response.sendRedirect(request.getContextPath() + "/users");
        } catch (SQLException | NumberFormatException e) {
            request.setAttribute("error", "Không thêm được User. Transaction đã rollback nếu gặp lỗi SQL: " + e.getMessage());
            request.getRequestDispatcher("/user/create.jsp").forward(request, response);
        }
    }
}
```

### `src/main/webapp/index.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<% response.sendRedirect(request.getContextPath() + "/users"); %>
```

### `src/main/webapp/user/create.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<!DOCTYPE html><html lang="vi"><head><meta charset="UTF-8"><title>Thêm User và phân quyền</title></head>
<body style="font-family:Arial;max-width:650px;margin:40px auto">
<h2>Thêm User (JDBC Transaction)</h2>
<% if (request.getAttribute("error") != null) { %><p style="color:red">Không thể lưu dữ liệu. Kiểm tra đầu vào hoặc Console/Tomcat để biết chi tiết.</p><% } %>
<form method="post" action="<%= request.getContextPath() %>/users">
<p>Họ tên: <input name="name" required maxlength="120"></p>
<p>Email: <input name="email" type="email" required maxlength="120"></p>
<p>Quốc gia: <input name="country" required maxlength="120"></p>
<p>Quyền hạn:</p>
<label><input type="checkbox" name="permissions" value="1"> Thêm (Add)</label>
<label><input type="checkbox" name="permissions" value="2"> Sửa (Edit)</label>
<label><input type="checkbox" name="permissions" value="3"> Xóa (Delete)</label>
<label><input type="checkbox" name="permissions" value="4"> Xem (View)</label>
<p><button type="submit">Lưu thông tin</button> <a href="<%= request.getContextPath() %>/users">Danh sách</a></p>
</form></body></html>
```

### `src/main/webapp/user/list.jsp`

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="com.codegym.model.User,java.util.List" %>
<!DOCTYPE html><html lang="vi"><head><meta charset="UTF-8"><title>Danh sách User</title></head><body style="font-family:Arial;max-width:850px;margin:40px auto">
<h2>Danh sách User</h2><p><a href="<%= request.getContextPath() %>/users?action=create">+ Thêm User và phân quyền</a></p>
<table border="1" cellpadding="8" cellspacing="0"><tr><th>ID</th><th>Tên</th><th>Email</th><th>Quốc gia</th></tr>
<% for (User u : (List<User>) request.getAttribute("users")) { %>
<tr><td><%= u.getId() %></td><td><%= u.getName().replace("&","&amp;").replace("<","&lt;").replace(">","&gt;").replace(""","&quot;") %></td><td><%= u.getEmail().replace("&","&amp;").replace("<","&lt;").replace(">","&gt;") %></td><td><%= u.getCountry().replace("&","&amp;").replace("<","&lt;").replace(">","&gt;") %></td></tr>
<% } %></table></body></html>
```

### `src/main/webapp/WEB-INF/web.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="6.0"><display-name>User Management Transaction</display-name></web-app>
```

### `.gitignore`

```text
target/
.idea/
.vscode/
*.iml
.env
```

### `README.md`

```text
# JDBC Transaction – Thêm User và phân quyền

Dự án Maven WAR dùng Java 17, Tomcat 10.1+, JSP/Servlet Jakarta, JDBC và MySQL 8.

## Chạy thử
1. Chạy `sql/demo.sql` trong MySQL (sử dụng database `demo`; yêu cầu bảng `users` có cột id AUTO_INCREMENT, name, email, country).
2. Thiết lập biến môi trường `DB_URL`, `DB_USER`, `DB_PASSWORD` phù hợp MySQL đang dùng. Ví dụ PowerShell: `$env:DB_PASSWORD='mat_khau_cua_ban'`.
3. Chạy `mvn clean package`.
4. Copy `target/user-management.war` vào thư mục `webapps` của Tomcat 10.1+, sau đó khởi động Tomcat.
5. Mở `http://localhost:8080/user-management/users?action=create`.

## Kiểm thử
- Thành công: nhập User và chọn quyền 1, 4. Kiểm tra `SELECT * FROM users ORDER BY id DESC;` và `SELECT * FROM user_permission;`.
- Rollback: chọn quyền không tồn tại (chẳng hạn sửa thủ công value của checkbox thành 999 qua DevTools) để vi phạm khóa ngoại. Kiểm tra không có User mới được lưu.
- Mọi bảng phải dùng InnoDB để hỗ trợ transaction.

## GitHub
Tạo repository trống `user-permission-transaction`, sau đó chạy:
```powershell
git init
git add .
git commit -m "feat: JDBC transaction user permissions"
git branch -M main
git remote add origin https://github.com/USERNAME/user-permission-transaction.git
git push -u origin main
```
Thay USERNAME bằng tên GitHub của bạn. Nộp URL repository.
```

## Ghi chú nộp bài
Dùng cấu trúc dự án riêng ở trên để push lên GitHub, không chỉ upload một file Markdown. Dự án chưa được kiểm thử bằng Tomcat/MySQL thực tế.
