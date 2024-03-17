# Java/Spring Web项目请求日志打印的4种实现

> [SpringMVC之DispatcherServlet（一）](https://blog.csdn.net/sinat_34976604/article/details/87090623)
[Java过滤器—Filter用法简介](https://blog.csdn.net/wcc27857285/article/details/80298342)
[Spring 拦截器和过滤器的区别？](https://www.zhihu.com/question/30212464/answer/1786967139)

## 前言
今天，一个同事问道，怎么给我们的 Spring Web 服务打印请求日志？他主要困惑的地方是，我们的服务提供了Restful风格的API，请求方法/请求参数格式多样化，不知道如果处理比较好。
听到这个问题，我一时想起了大学写的几个用例，本想翻一下发给他，发现都是只有代码没有想过描述。算了算了，补一个文档吧，故有此篇。
统一日志打印，一般想到的是用面向切面的方式实现。常见的实现方式有，过滤器 Filter、拦截器 Intercprot、增强 DispatcherServlet、Spring Aop。
对于，过滤器、拦截器、Servlet 执行顺序如下图：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240317194905.png#id=jbkKO&originHeight=450&originWidth=568&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 示例项目仓库

- [Github项目路径 : https://github.com/eden2f/springboot-web-demo](https://github.com/eden2f/springboot-web-demo)
- [Gitee项目路径 : https://gitee.com/eden2f/springboot-web-demo](https://gitee.com/eden2f/springboot-web-demo)
## 过滤器 Filter
Filter译为过滤器。 由于 Servlet 规范是开放的，借助于公众与开源社区的力量， Servlet 规范越来越科学，功能也越来越强大。 2000 年， Sun 公司在 Servlet2.3 规范中添加了 Filter 功能，并在 Servlet2.4 中对 Filter 进行了细节上的补充。
当客户端向服务器端发送一个请求时，如果有对应的过滤器进行拦截，过滤器可以改变请求的内容、或者重新设置请求协议的相关信息等，然后再将请求发送给服务器端的Servlet进行处理。当Servlet对客户端做出响应时，过滤器同样可以进行拦截，将响应内容进行修改或者重新设置后，再响应给客户端浏览器。在上述过程中，客户端与服务器端并不需要知道过滤器的存在。
在一个Web应用程序中，可以部署多个过滤器进行拦截，这些过滤器组成了一个过滤器链（设计模式：责任链模式的应用）。过滤器链中的每个过滤器负责特定的操作和任务，客户端的请求在这些过滤器之间传递，直到服务器端的Servlet。具体执行流程如下：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240317194931.png#id=vKhnv&originHeight=475&originWidth=1502&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
所以， 对于请求处理来说， 过滤器是一个天然的切面。
```java
@Slf4j
@Component
@WebFilter(filterName = "logFilter", urlPatterns = "/*")
public class LogFilter implements Filter {


    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper((HttpServletRequest) request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper((HttpServletResponse) response);
        RequestLogInfo requestLogInfo = HttpLoggingUtil.initByHttpServletRequest(requestWrapper);
        chain.doFilter(requestWrapper, responseWrapper);
        HttpLoggingUtil.updateByHttpServletResponse(requestLogInfo, requestWrapper, responseWrapper);
        log.info(JSON.toJSONString(requestLogInfo));
    }

    @Override
    public void destroy() {

    }
}
```
## 拦截器 Intercprot
过滤器和拦截器 底层实现方式大不相同，过滤器 是基于函数回调的，拦截器 则是基于Java的反射机制（动态代理）实现的。
过滤器实现的是 javax.servlet.Filter 接口，而这个接口是在 Servlet 规范中定义的，也就是说过滤器 Filter 的使用要依赖于 Tomcat 等容器，导致它只能在 web 程序中使用。而拦截器(Interceptor) 它是一个 Spring 组件，并由 Spring 容器管理，并不依赖 Tomcat 等容器，是可以单独使用的。不仅能应用在 web 程序中，也可以用于Application、Swing 等程序中。
过滤器Filter是在请求进入容器后，但在进入servlet之前进行预处理，请求结束是在servlet处理完以后。拦截器 Interceptor 是在请求进入servlet后，在进入Controller之前进行预处理的，Controller 中渲染了对应的视图之后请求结束。
过滤器几乎可以对所有进入容器的请求起作用，而拦截器只会对Controller中请求或访问static目录下的资源请求起作用。
```java
@Slf4j
@Component
public class LogInterceptor extends HandlerInterceptorAdapter {

    private final ThreadLocal<Long> startTimeThreadLocal = new ThreadLocal<>();
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) {
        startTimeThreadLocal.set(System.currentTimeMillis());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        super.postHandle(request, response, handler, modelAndView);
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
        RequestLogInfo requestLogInfo = HttpLoggingUtil.initByHttpServletRequest(requestWrapper);
        requestLogInfo.setCosTimeMillis(startTimeThreadLocal.get());
        HttpLoggingUtil.updateByHttpServletResponse(requestLogInfo, requestWrapper, responseWrapper);
        log.info(JSON.toJSONString(requestLogInfo));
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response, Object handler, Exception ex) throws Exception {
        super.afterCompletion(request, response, handler, ex);
        startTimeThreadLocal.remove();
    }
}
```
## 增强 DispatcherServlet
DispatcherServlet 继承自 HttpServlet ，通过使用 Servlet API 对 HTTP 请求进行响应。其工作大致分为两个部分：一是初始化部分，由 init() 启动，经 initServletBean() ， 通过 initWebApplicationContext() 最终调用 DispatcherServlet 的 initStrategies 方法，在该方法里对诸如 handlerMapping，ViewResolver 等进行初始化；另一个是对 HTTP 请求进行响应的部分，作为一个 Servlet，Web 容器会调用其 doGet()，doPost 等方法，经过 FrameworkServlet 的processRequest() 简单处理后会调用 DispatcherServlet 的doService() 方法，该方法会调用 doDispatch()，doDispatch 是实现 MVC 模式的主要部分，流程图如下
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240317194953.png#id=uKqZa&originHeight=635&originWidth=968&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
所以，利用他的特点，我们可以修饰 DispatcherServlet ，给其增加统一日志打印能力。
```
@Slf4j
@Component(value = DispatcherServletAutoConfiguration.DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public class LoggableDispatcherServlet extends DispatcherServlet {

    @Override
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
        RequestLogInfo requestLogInfo = HttpLoggingUtil.initByHttpServletRequest(requestWrapper);
        try {
            super.doDispatch(requestWrapper, responseWrapper);
        } finally {
            HttpLoggingUtil.updateByHttpServletResponse(requestLogInfo, requestWrapper, responseWrapper);
            log.info(JSON.toJSONString(requestLogInfo));
        }
    }
}
```
## Spring Aop 切面编程
动态代理的实现，给切点做增强，可以用环绕通知@Around，也可以用前置通知@Before+后置通知@AfterReturning，完成日志打印功能。
```java
@Slf4j
@Aspect
@Component
public class WebLogAspect {

    private final ThreadLocal<RequestLogInfo> requestInfoThreadLocal = new ThreadLocal<>();

    @Pointcut("execution(public * com.eden.springbootwebdemo.controller..*.*(..))")
    public void webLog() {

    }

    @Before(value = "webLog()")
    public void doBefore(JoinPoint point) {
        Long startTime = System.currentTimeMillis();
        ServletRequestAttributes attributes =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        RequestLogInfo requestLogInfo = new RequestLogInfo();
        requestLogInfo.setCosTimeMillis(startTime);
        requestLogInfo.setRemoteAddr(request.getRemoteAddr());
        requestLogInfo.setRequestUri(request.getRequestURL().toString());
        requestLogInfo.setMethod(point.getSignature().getDeclaringTypeName() + "." + point.getSignature().getName());
        requestLogInfo.setRequest(Arrays.toString(point.getArgs()));
        requestInfoThreadLocal.set(requestLogInfo);
    }

    @AfterReturning(value = "webLog()", returning = "ret")
    public void doAferReturning(Object ret) {
        RequestLogInfo requestLogInfo = requestInfoThreadLocal.get();
        requestLogInfo.setCosTimeMillis(System.currentTimeMillis() - requestLogInfo.getCosTimeMillis());
        requestLogInfo.setResponse(ret);
        log.info(JSON.toJSONString(requestLogInfo));
        requestInfoThreadLocal.remove();
    }

}
```
## 本文其他工具类
RequestLogInfo
```java
@Setter
@Getter
@ToString
@NoArgsConstructor
@AllArgsConstructor
public class RequestLogInfo {

    /**
     * 请求资源
     */
    private String requestUri;

    /**
     * 调用者地址
     */
    private String remoteAddr;

    /**
     * 请求头
     */
    private Object requestHeaders;

    /**
     * 被调方法
     */
    private String method;

    /**
     * 请求参数
     */
    private Object request;

    /**
     * 响应状态
     */
    private Integer status;

    /**
     * 响应头
     */
    private Object responseHeaders;

    /**
     * 响应数据
     */
    private Object response;

    /**
     * 接口耗时
     */
    private Long cosTimeMillis;

}
```
HttpLoggingUtil.class
```java
public class HttpLoggingUtil {

    public static RequestLogInfo initByHttpServletRequest(ContentCachingRequestWrapper requestWrapper) {
        RequestLogInfo requestLogInfo = new RequestLogInfo();
        requestLogInfo.setCosTimeMillis(System.currentTimeMillis());
        requestLogInfo.setRequestUri(requestWrapper.getRequestURI());
        requestLogInfo.setRemoteAddr(requestWrapper.getRemoteAddr());
        requestLogInfo.setRequestHeaders(getRequestHeaders(requestWrapper));
        return requestLogInfo;
    }

    public static void updateByHttpServletResponse(RequestLogInfo requestLogInfo, ContentCachingRequestWrapper requestWrapper, ContentCachingResponseWrapper responseWrapper) throws IOException {
        String method = requestWrapper.getMethod();
        requestLogInfo.setMethod(method);
        if (method.equals(RequestMethod.GET.name())) {
            requestLogInfo.setRequest(requestWrapper.getParameterMap());
        } else {
            requestLogInfo.setResponse(new String(requestWrapper.getContentAsByteArray()));
        }
        requestLogInfo.setStatus(responseWrapper.getStatus());
        requestLogInfo.setResponse(new String(responseWrapper.getContentAsByteArray()));
        responseWrapper.copyBodyToResponse();
        requestLogInfo.setResponseHeaders(getResponsetHeaders(responseWrapper));
        requestLogInfo.setCosTimeMillis(System.currentTimeMillis() - requestLogInfo.getCosTimeMillis());
    }

    private static Map<String, Object> getResponsetHeaders(ContentCachingResponseWrapper response) {
        Map<String, Object> headers = new HashMap<>(16);
        Collection<String> headerNames = response.getHeaderNames();
        for (String headerName : headerNames) {
            headers.put(headerName, response.getHeader(headerName));
        }
        return headers;
    }

    private static Map<String, Object> getRequestHeaders(HttpServletRequest request) {
        Map<String, Object> headers = new HashMap<>(16);
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String headerName = headerNames.nextElement();
            headers.put(headerName, request.getHeader(headerName));
        }
        return headers;
    }
}
```
