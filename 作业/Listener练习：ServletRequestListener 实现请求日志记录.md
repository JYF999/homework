# 请求日志记录实现

## 代码实现

### 1. RequestLoggingListener.java

```java
package com.example.listener;

import javax.servlet.*;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpServletRequest;
import java.time.Instant;
import java.time.Duration;
import java.util.logging.Logger;

/**
 * 监听每个 HTTP 请求并记录详细信息的监听器
 */
@WebListener
public class RequestLoggingListener implements ServletRequestListener {

    // 创建日志记录器
    private static final Logger logger = Logger.getLogger(RequestLoggingListener.class.getName());

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        // 获取 HttpServletRequest 对象
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();

        // 记录请求的开始时间
        Instant startTime = Instant.now();
        request.setAttribute("startTime", startTime);

        // 获取请求的详细信息
        String clientIP = request.getRemoteAddr();
        String method = request.getMethod();
        String uri = request.getRequestURI();
        String queryString = request.getQueryString();
        String userAgent = request.getHeader("User-Agent");

        // 格式化日志信息
        String logMessage = String.format(
                "Request Initialized - Time: %s, IP: %s, Method: %s, URI: %s%s, User-Agent: %s",
                startTime,
                clientIP,
                method,
                uri,
                queryString == null ? "" : "?" + queryString,
                userAgent
        );

        // 记录日志
        logger.info(logMessage);
    }

    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        // 获取 HttpServletRequest 对象
        HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();

        // 获取开始时间和结束时间
        Instant startTime = (Instant) request.getAttribute("startTime");
        Instant endTime = Instant.now();

        // 计算请求处理时间
        long processingTime = Duration.between(startTime, endTime).toMillis();

        // 获取请求的 URI
        String uri = request.getRequestURI();

        // 格式化日志信息
        String logMessage = String.format(
                "Request Destroyed - URI: %s, Processing Time: %d ms",
                uri,
                processingTime
        );

        // 记录日志
        logger.info(logMessage);
    }
}
```

### 2. TestServlet.java

```java
package com.example.servlet;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.*;
import java.io.IOException;

/**
 * 用于测试 RequestLoggingListener 的 Servlet
 */
@WebServlet("/test")
public class TestServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 模拟一些处理时间
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        resp.setContentType("text/plain");
        resp.getWriter().write("TestServlet is running...");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 调用 doGet 方法处理 POST 请求
        doGet(req, resp);
    }
}
```

## 实现说明

### 功能概述

`RequestLoggingListener` 实现了 `ServletRequestListener` 接口，用于监听每个 HTTP 请求的开始和结束事件。在请求初始化时，记录请求的详细信息和开始时间；在请求销毁时，计算并记录请求的处理时间。此功能有助于监控应用的性能，分析请求的行为，为优化提供数据支持。

### 核心实现步骤

1. **创建监听器类**
   - 使用 `@WebListener` 注解，将 `RequestLoggingListener` 注册为监听器。
   - 实现 `requestInitialized` 和 `requestDestroyed` 方法，分别在请求初始化和销毁时执行。
2. **记录请求初始化信息**
   - **请求开始时间**：在 `requestInitialized` 方法中，使用 `Instant.now()` 获取请求开始时间，并存储在请求属性中，以便在请求结束时计算处理时间。
   - **请求详细信息**：获取客户端 IP、请求方法、请求 URI、查询字符串和 User-Agent 等信息。
   - **日志记录**：使用 `Logger` 将格式化的日志信息记录下来，便于后续分析。
3. **记录请求结束信息**
   - **请求处理时间**：在 `requestDestroyed` 方法中，获取开始时间和结束时间，计算处理时间（毫秒）。
   - **日志记录**：记录请求的 URI 和处理时间，帮助识别处理较慢的请求。
4. **日志格式**
   - 采用统一且清晰的日志格式，包含所有关键信息，便于阅读和分析。
   - 在请求开始和结束时分别记录不同的日志信息，提供完整的请求生命周期数据。

### 额外功能

1. **模拟处理时间**
   - 在 `TestServlet` 中，使用 `Thread.sleep(100)` 模拟处理延迟，便于测试请求处理时间的记录功能。
   - 通过模拟实际的处理时间，可以更准确地测试监听器的功能。
2. **日志输出优化**
   - **时间处理**：使用 `Instant` 和 `Duration` 类处理时间，更精确和现代化，避免了使用过时的 `Date` 类。
   - **完整 URI 记录**：在日志中包含完整的请求 URI（包括查询字符串），提供更详细的请求信息。
   - **异常处理**：在模拟处理时间时，处理 `InterruptedException`，确保线程中断状态被正确设置。
3. **扩展性**
   - **支持多种请求方法**：监听器支持 `GET`、`POST` 等所有类型的 HTTP 请求。
   - **可扩展的日志信息**：可以根据需要，增加更多的请求信息记录，如 `Referer`、`Session ID` 等。

## 测试说明

1. **部署应用**

   - 将上述代码编译并部署到支持 Servlet 3.0 及以上版本的应用服务器（如 Tomcat 9 或更高版本）。

2. **访问测试 Servlet**

   - 在浏览器中访问 `http://localhost:8080/yourapp/test`，可添加查询参数，例如 `?param=value`，以测试查询字符串的记录。

3. **查看日志输出**

   - 检查应用服务器的日志，确认请求的详细信息和处理时间被正确记录。

   **示例日志输出：**

   ```java
   Oct 02, 2023 16:00:00 PM com.example.listener.RequestLoggingListener requestInitialized
   INFO: Request Initialized - Time: 2023-10-02T08:00:00Z, IP: 127.0.0.1, Method: GET, URI: /yourapp/test?param=value, User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
   Oct 02, 2023 16:00:00 PM com.example.listener.RequestLoggingListener requestDestroyed
   INFO: Request Destroyed - URI: /yourapp/test, Processing Time: 105 ms
   ```

