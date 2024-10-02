# LoginFilter.java

```java
java复制代码package com.example.filter;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

/**
 * 登录验证过滤器
 * 
 * 增加功能：
 * 1. 将未登录用户的原始访问路径存储在 session 中，以便登录后可以重定向回该路径。
 * 2. 增加对静态资源的自动过滤，如 .css、.js、.png 等无需登录即可访问的文件类型。
 */
@WebFilter("/*")
public class LoginFilter implements Filter {

    // 定义不需要登录即可访问的路径列表
    private static final List<String> EXCLUDED_PATHS = Arrays.asList(
        "/login",
        "/register",
        "/public/"
    );

    // 静态资源扩展名列表
    private static final List<String> STATIC_RESOURCES = Arrays.asList(
        ".css", ".js", ".png", ".jpg", ".jpeg", ".gif", ".woff", ".woff2", ".ttf", ".eot"
    );

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 过滤器初始化方法，可在此执行初始化操作
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // 将 ServletRequest 和 ServletResponse 转换为 HttpServletRequest 和 HttpServletResponse
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // 获取请求的URI
        String uri = httpRequest.getRequestURI();
        // 获取上下文路径
        String contextPath = httpRequest.getContextPath();

        // 检查请求路径是否在排除列表中，或者是否是静态资源
        if (isExcludedPath(uri, contextPath) || isStaticResource(uri)) {
            // 如果在排除列表中或者是静态资源，直接放行
            chain.doFilter(request, response);
            return;
        }

        // 获取会话对象
        HttpSession session = httpRequest.getSession(false);

        // 检查用户是否已登录（会话中是否存在"user"属性）
        if (session != null && session.getAttribute("user") != null) {
            // 用户已登录，允许请求继续
            chain.doFilter(request, response);
        } else {
            // 用户未登录，将当前请求路径存入 session，并重定向到登录页面
            HttpSession newSession = httpRequest.getSession(true);
            newSession.setAttribute("redirectAfterLogin", uri);  // 存储原始路径
            httpResponse.sendRedirect(contextPath + "/login");
        }
    }

    @Override
    public void destroy() {
        // 过滤器销毁方法，可在此执行资源释放等操作
    }

    /**
     * 检查请求路径是否在排除列表中
     * @param uri 请求的URI
     * @param contextPath 上下文路径
     * @return 如果在排除列表中，返回 true，否则返回 false
     */
    private boolean isExcludedPath(String uri, String contextPath) {
        // 去除上下文路径，得到实际请求路径
        String path = uri.substring(contextPath.length());

        // 检查路径是否以排除列表中的任何一个路径开始
        for (String exclude : EXCLUDED_PATHS) {
            if (path.startsWith(exclude)) {
                return true;
            }
        }
        return false;
    }

    /**
     * 检查请求是否为静态资源
     * @param uri 请求的URI
     * @return 如果是静态资源，返回 true，否则返回 false
     */
    private boolean isStaticResource(String uri) {
        // 遍历静态资源列表，检查是否以指定扩展名结尾
        for (String extension : STATIC_RESOURCES) {
            if (uri.endsWith(extension)) {
                return true;
            }
        }
        return false;
    }
}
```

# 实现说明文档

### 1. **类的创建**

`LoginFilter` 类继承了 `javax.servlet.Filter` 接口，用于处理所有的 URL 请求，确保未登录用户无法访问受保护的资源。如果用户未登录，则重定向到登录页面，并在登录后将用户重定向回原始访问页面。

### 2. **功能扩展**

#### a. **支持静态资源过滤**

增加了 `isStaticResource` 方法来处理静态资源的自动过滤。常见的静态文件类型如 `.css`、`.js`、`.png`、`.jpg` 等无需验证用户登录状态即可访问。静态资源列表如下：

```java
private static final List<String> STATIC_RESOURCES = Arrays.asList(
    ".css", ".js", ".png", ".jpg", ".jpeg", ".gif", ".woff", ".woff2", ".ttf", ".eot"
);
```

`isStaticResource(String uri)` 方法检查当前请求的 URI 是否是静态资源。这样可以避免不必要的登录验证，优化资源加载速度。

#### b. **登录后重定向到原始访问路径**

增加了对未登录用户的优化处理。如果用户在访问某个受保护资源时未登录，系统会将当前的请求路径保存到会话中，并在用户成功登录后重定向回原始页面。具体流程如下：

1. 如果用户未登录，获取当前的 URI，并将其存储在 `session` 中的 `redirectAfterLogin` 属性中。
2. 用户登录成功后，可以通过该属性获取之前访问的路径并进行重定向，提升用户体验。

示例代码：

```java
HttpSession newSession = httpRequest.getSession(true);
newSession.setAttribute("redirectAfterLogin", uri);  // 存储原始路径
```

### 3. **过滤器的基本逻辑**

#### a. **`@WebFilter` 注解**

通过 `@WebFilter("/*")` 注解将过滤器应用到所有 URL 路径，保证所有的请求都必须通过该过滤器进行处理。

#### b. **核心过滤逻辑：`doFilter` 方法**

- **转换请求对象** 将 `ServletRequest` 和 `ServletResponse` 分别转换为 `HttpServletRequest` 和 `HttpServletResponse`，以便使用 HTTP 专有的方法和属性。
- **排除特定路径和静态资源** 调用 `isExcludedPath()` 方法，检查请求路径是否属于无需登录的路径；调用 `isStaticResource()` 检查是否为静态资源。如果路径匹配，直接放行请求。
- **会话检查** 获取当前请求的会话对象 `HttpSession`，检查其中是否存在 `user` 属性。如果用户已登录，放行请求；如果未登录，存储原始访问路径并重定向到登录页面。

### 4. **排除列表与静态资源过滤**

- **排除列表**
  包含无需登录即可访问的路径，主要用于登录、注册和公共资源等：

  ```java
  private static final List<String> EXCLUDED_PATHS = Arrays.asList(
      "/login",
      "/register",
      "/public/"
  );
  ```

- **静态资源过滤**
  处理常见的静态文件资源（如 `.css`, `.js`, `.png` 等），避免对这些资源进行不必要的登录检查，提高系统性能。

### 5. **初始化与销毁方法**

- `init()`：过滤器初始化时执行，可用于资源初始化。
- `destroy()`：在过滤器销毁时释放资源。