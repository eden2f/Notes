# Spring集成SpringMVC(构建Web应用程序)

### Spring MVC 的请求处理流程

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240216134647.png)

### 示例环境
1. JDK 1.8
2. Tomcat 8
3. Spring 4.0.0.RELEASE
4. IntelliJ IDEA 2017
5. Gradle 4.6
6. OS: ubuntu18.04

### jar包依赖

``` xml
group 'com.xxx'
version '1.0-SNAPSHOT'

apply plugin: 'java'
apply plugin: 'war'

sourceCompatibility = 1.8

repositories {
    mavenLocal()
    maven {url 'http://maven.aliyun.com/nexus/content/groups/public/'}
    mavenCentral()
}

dependencies {
    compile group: 'org.springframework', name: 'spring-aop', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-aspects', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-beans', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-context', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-context-support', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-core', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-expression', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-instrument', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-instrument-tomcat', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-jdbc', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-jms', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-messaging', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-orm', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-oxm', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-test', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-tx', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-web', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-webmvc', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-webmvc-portlet', version: '4.0.0.RELEASE'
    compile group: 'org.springframework', name: 'spring-websocket', version: '4.0.0.RELEASE'
    compile group: 'org.aspectj', name: 'aspectjweaver', version: '1.8.7'


    compile group: 'javax.el', name: 'javax.el-api', version: '2.2.4'
    compile group: 'javax.validation', name: 'validation-api', version: '1.1.0.Final'
    compile group: 'org.hibernate', name: 'hibernate-validator', version: '5.2.2.Final'


    compile group: 'javax.servlet', name: 'javax.servlet-api', version: '3.1.0'
    compile group: 'taglibs', name: 'standard', version: '1.1.2'
    compile group: 'jstl', name: 'jstl', version: '1.2'

    testCompile group: 'junit', name: 'junit', version: '4.12'
    testCompile group: 'org.mockito', name: 'mockito-all', version: '1.10.19'
}
```

### 搭建 Spring MVC

1. 配置 DispatcherServlet

注意：按照传统方式，想 DispatcherServlet 这样的 Servlet 会配置在 web.xml 文件中。当然，这是配置DispatcherServlet的方法之一。 但是借助于Servlet 3 规范和 Spring 3.1 的功能增强，这种方式已经不是唯一方案。本文案例使用 Java 将 DispatcherServlet 配置在 Servlet 容器中， 而不会在使用 web.xml 文件。

``` java
/**
 * 配置 DispatcherServlet
 * @author aspen
 * @date Created in 2018/6/4
 */
public class SpittrWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer{

    /**
     * 根容器
     * @return
     */
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{ RootConfig.class};
    }

    /**
     * springMVC 容器
     * @return
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{ WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```
2. 启用 Spring MVC
``` java
@Configuration
@EnableWebMvc  // 启用 Spring MVC
@ComponentScan("com.aspen.springlearning")  // 启用组件扫描
public class WebConfig extends WebMvcConfigurerAdapter{

    @Bean
    public ViewResolver viewResoler(){
        // 配置 JSP 视图解析器
        InternalResourceViewResolver resourceViewResolver = new InternalResourceViewResolver();
        resourceViewResolver.setPrefix("/WEB-INF/views/");
        resourceViewResolver.setSuffix(".jsp");
        resourceViewResolver.setExposeContextBeansAsAttributes(true);
        return resourceViewResolver;
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer){
        // 配置静态资源的处理
        configurer.enable();
    }
}
```
``` java
@Configuration
@ComponentScan(basePackages={"com.aspen.springlearning"},
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)
    }
)
public class RootConfig {
}
```  

3. 编写基本的控制器
``` java
@Controller
@RequestMapping(value = {"/", "/homepage"})
public class HomeController {

    @RequestMapping(method = RequestMethod.GET, produces = MediaType.TEXT_HTML_VALUE)
    public String home(){
        return "home";
    }
}
```

4. 编写程序主页 home.jsp

``` html
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Home</title>
</head>
<body>
    <h1>Welcome </h1>

</body>
</html>
```

至此，一个由 Spring MVC 搭建的 Web应用程序已经完成。下图是代码存储结构，在运行之前，先为控制器编写单元测试，验证一下。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240216134913.png)

1. 编写单元测试

至此，一个由 Spring MVC 搭建的 Web应用程序已经完成。在运行之前，先为控制器编写单元测试，验证一下。

``` java
public class HomeControllerTest {

    HomeController homeController = null;

    @Before
    public void setUp(){
        homeController = new HomeController();
    }

    @Test
    public void testHomePage() throws Exception {
        // 搭建 MockMvc
        MockMvc mockMvc = standaloneSetup(homeController).build();
        // 1. 对 "/" 执行GET请求"
        // 2. 预期得到 home 视图
        mockMvc.perform(get("/")).andExpect(view().name("home"));
    }

}
```

6. 运行程序

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240216134948.png)





  
 
