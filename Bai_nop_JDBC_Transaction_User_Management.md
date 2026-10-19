# BÀI NỘP: JDBC TRANSACTION – COMMIT VÀ ROLLBACK

Dự án mẫu đầy đủ (Java 17 + Servlet + JSP + JDBC + MySQL). Tạo các file đúng theo đường dẫn dưới đây.

## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
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

## sql/schema.sql

```sql
CREATE DATABASE IF NOT EXISTS user_management CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE user_management;
CREATE TABLE IF NOT EXISTS Employee (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  salary DECIMAL(12,2) NOT NULL,
  created_at DATETIME NOT NULL
) ENGINE=InnoDB;
-- Chỉ để thử với bảng trống: TRUNCATE TABLE Employee; (lệnh này không rollback được).
SELECT * FROM Employee;
```

## src/main/java/com/codegym/model/Employee.java

```java
package com.codegym.model;
import java.math.BigDecimal;
import java.time.LocalDateTime;
public class Employee {
    private final long id;
    private final String name;
    private final BigDecimal salary;
    private final LocalDateTime createdAt;
    public Employee(long id, String name, BigDecimal salary, LocalDateTime createdAt) {
        this.id=id; this.name=name; this.salary=salary; this.createdAt=createdAt;
    }
    public long getId(){return id;}
    public String getName(){return name;}
    public BigDecimal getSalary(){return salary;}
    public LocalDateTime getCreatedAt(){return createdAt;}
}
```

## src/main/java/com/codegym/dao/IUserDAO.java

```java
package com.codegym.dao;
import com.codegym.model.Employee;
import java.sql.SQLException;
import java.util.List;
public interface IUserDAO {
    void insertUpdateUseTransaction() throws SQLException;
    void insertUpdateUseTransaction(boolean simulateFailure) throws SQLException;
    List<Employee> getEmployees() throws SQLException;
}
```

## src/main/java/com/codegym/dao/UserDAO.java

```java
package com.codegym.dao;

import com.codegym.model.Employee;
import java.math.BigDecimal;
import java.sql.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

public class UserDAO implements IUserDAO {
    private static final String INSERT_SQL = "INSERT INTO Employee(name, salary, created_at) VALUES (?, ?, ?)";
    private static final String UPDATE_SQL = "UPDATE Employee SET salary = ? WHERE name = ?";
    private static final String SELECT_SQL = "SELECT id, name, salary, created_at FROM Employee ORDER BY id";

    private Connection getConnection() throws SQLException {
        String url = System.getenv().getOrDefault("DB_URL", "jdbc:mysql://localhost:3306/user_management?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Bangkok");
        String user = System.getenv().getOrDefault("DB_USER", "root");
        String password = System.getenv().getOrDefault("DB_PASSWORD", "");
        return DriverManager.getConnection(url, user, password);
    }

    @Override
    public void insertUpdateUseTransaction() throws SQLException {
        insertUpdateUseTransaction(false);
    }

    @Override
    public void insertUpdateUseTransaction(boolean simulateFailure) throws SQLException {
        try (Connection conn = getConnection()) {
            boolean oldAutoCommit = conn.getAutoCommit();
            conn.setAutoCommit(false);
            try (PreparedStatement psInsert = conn.prepareStatement(INSERT_SQL);
                 PreparedStatement psUpdate = conn.prepareStatement(UPDATE_SQL)) {
                insert(psInsert, "Quynh", new BigDecimal("10.00"));
                insert(psInsert, "Ngan", new BigDecimal("20.00"));
                if (simulateFailure) {
                    // Cố tình KHÔNG gán tham số 1 để quan sát rollback
                    psUpdate.setBigDecimal(2, new BigDecimal("999.99"));
                    psUpdate.setString(2, "Quynh");
                } else {
                    psUpdate.setBigDecimal(1, new BigDecimal("999.99"));
                    psUpdate.setString(2, "Quynh");
                }
                psUpdate.executeUpdate();
                conn.commit();
            } catch (SQLException | RuntimeException e) {
                try { conn.rollback(); }
                catch (SQLException rollbackError) { e.addSuppressed(rollbackError); }
                throw e;
            } finally {
                try { conn.setAutoCommit(oldAutoCommit); }
                catch (SQLException ignored) { /* Connection is about to close */ }
            }
        }
    }

    private void insert(PreparedStatement ps, String name, BigDecimal salary) throws SQLException {
        ps.setString(1, name);
        ps.setBigDecimal(2, salary);
        ps.setTimestamp(3, Timestamp.valueOf(LocalDateTime.now()));
        ps.executeUpdate();
    }

    @Override
    public List<Employee> getEmployees() throws SQLException {
        List<Employee> employees = new ArrayList<>();
        try (Connection conn = getConnection();
             PreparedStatement ps = conn.prepareStatement(SELECT_SQL);
             ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                employees.add(new Employee(rs.getLong("id"), rs.getString("name"),
                    rs.getBigDecimal("salary"), rs.getTimestamp("created_at").toLocalDateTime()));
            }
        }
        return employees;
    }
}
```

## src/main/java/com/codegym/web/UserServlet.java

```java
package com.codegym.web;

import com.codegym.dao.IUserDAO;
import com.codegym.dao.UserDAO;
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
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setCharacterEncoding("UTF-8");
        String action = request.getParameter("action");
        if ("test-use-tran".equals(action)) {
            testUseTran(request, response);
        } else {
            showList(request, response);
        }
    }

    // Bài thực hành yêu cầu GET; thực tế hệ thống nên dùng POST cho thao tác ghi dữ liệu.
    private void testUseTran(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        boolean fail = "true".equalsIgnoreCase(request.getParameter("fail"));
        try {
            userDAO.insertUpdateUseTransaction(fail);
            request.setAttribute("message", "COMMIT thành công: đã chèn 2 nhân viên và cập nhật Quynh.");
        } catch (SQLException e) {
            getServletContext().log("Transaction thất bại, đã yêu cầu ROLLBACK", e);
            request.setAttribute("message", "ROLLBACK do lỗi: " + e.getMessage());
        }
        showList(request, response);
    }

    private void showList(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            request.setAttribute("employees", userDAO.getEmployees());
            request.getRequestDispatcher("/WEB-INF/views/list.jsp").forward(request, response);
        } catch (SQLException e) {
            throw new ServletException("Không truy vấn được bảng Employee. Kiểm tra MySQL và cấu hình DB.", e);
        }
    }
}
```

## src/main/webapp/index.jsp

```xml
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<% response.sendRedirect(request.getContextPath() + "/users"); %>
```

## src/main/webapp/WEB-INF/views/list.jsp

```xml
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page import="java.util.List,com.codegym.model.Employee" %>
<!doctype html>
<html lang="vi"><head><meta charset="UTF-8"><title>JDBC Transaction Demo</title>
<style>body{font-family:Arial,sans-serif;max-width:850px;margin:40px auto;line-height:1.5}table{width:100%;border-collapse:collapse;margin-top:20px}td,th{border:1px solid #ddd;padding:10px;text-align:left}a{display:inline-block;margin-right:12px;padding:10px;background:#224a92;color:white;text-decoration:none;border-radius:5px}.msg{background:#f0f4fb;padding:12px;border-left:4px solid #224a92}</style></head>
<body><h1>Demo JDBC Transaction</h1>
<p>Nhấn <strong>Thử rollback</strong> để gây lỗi ở câu UPDATE, hoặc <strong>Thử commit</strong> để chạy thành công.</p>
<a href="<%= request.getContextPath() %>/users?action=test-use-tran&amp;fail=true">Thử rollback</a>
<a href="<%= request.getContextPath() %>/users?action=test-use-tran">Thử commit</a>
<% if (request.getAttribute("message") != null) { %>
<p class="msg"><%= ((String)request.getAttribute("message")).replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;") %></p>
<% } %>
<table><thead><tr><th>ID</th><th>Tên</th><th>Lương</th><th>Ngày tạo</th></tr></thead><tbody>
<% for (Employee employee : (List<Employee>)request.getAttribute("employees")) { %>
<tr><td><%= employee.getId() %></td><td><%= employee.getName() %></td><td><%= employee.getSalary() %></td><td><%= employee.getCreatedAt() %></td></tr>
<% } %>
</tbody></table></body></html>
```

## src/main/webapp/WEB-INF/web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd" version="6.0"><display-name>User Management Transaction</display-name></web-app>
```

## .gitignore

```powershell
target/
.idea/
.vscode/
*.iml
.env
```

## README.md

```markdown
# Bài thực hành JDBC Transaction – User Management

Ứng dụng Maven WAR, Java 17, Tomcat 10.1, MySQL (InnoDB), Jakarta Servlet.

## Chạy

1. Chạy script `sql/schema.sql` trên MySQL. Bảng **Employee** phải dùng InnoDB.
2. Đặt biến môi trường `DB_URL`, `DB_USER`, `DB_PASSWORD` (mặc định URL localhost và user root, mật khẩu trống; nên tự cấu hình mật khẩu).
3. Tại thư mục dự án chạy `mvn clean package`.
4. Copy `target/user-management.war` vào `webapps` của Tomcat 10.1 và khởi động Tomcat.
5. Mở `http://localhost:8080/user-management/users`.

## Kiểm chứng All-or-Nothing

**Trước khi thử**, ghi nhận kết quả `SELECT * FROM Employee;` (có thể xóa dữ liệu thử bằng `TRUNCATE TABLE Employee;` ở ngoài transaction, **không dùng TRUNCATE trong transaction**).

- Lỗi chủ động: `/users?action=test-use-tran&fail=true` (gán sai tham số cập nhật, SQL sẽ lỗi). Kiểm tra `SELECT * FROM Employee;`: không có thêm hai dòng mới.
- Thành công: `/users?action=test-use-tran` (tham số đúng), hai dòng Quynh, Ngan được chèn và Quynh được UPDATE lên `999.99` trong một commit.

Mỗi lần chạy commit thành công sẽ tạo thêm dữ liệu; để chạy lại demo từ đầu hãy dọn dữ liệu thử theo ý bạn.

**Lưu ý:** `conn.rollback()` được gọi rõ ràng trong `catch` — không phụ thuộc vào việc driver có tự rollback khi close hay không. `CREATE TABLE`, `DROP TABLE`, `TRUNCATE` trong MySQL có implicit commit, vì vậy tạo bảng riêng trước transaction. Route GET để khớp đề thực hành, nhưng phần mềm thực tế nên dùng POST cho thao tác thay đổi dữ liệu.

## Nộp GitHub

Tạo repository mới trên GitHub, chạy:

```powershell
git init
git add .
git commit -m "feat: demonstrate JDBC commit and rollback"
git branch -M main
git remote add origin https://github.com/USERNAME/user-management-transaction.git
git push -u origin main
```

Thay `USERNAME` bằng tên tài khoản GitHub và nộp URL repository. Không commit mật khẩu MySQL.
```
