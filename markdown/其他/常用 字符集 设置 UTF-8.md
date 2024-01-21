# 常用 字符集 设置 UTF-8

### 字符集处理: UTF-8
字符集主要涉及 2 个方面
文件本身的字符集（文件，数据库存储使用，返回给浏览器端的 html 内容）
程序中编码解码时候使用的字符集（如解析 http 请求的数据）
#### 为了防止乱码，我们规定：所有的字符集都用 UTF-8。

1. idea2017 里设置字符集
	file -> settings -> editor -> fileEncodings -> Global Encoding , Project Encoding 都设置成 utf-8
2. JSP 中字符集的设置
``` jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
```
3. HTML 中字符集的设置
``` html
<head>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8"/>
</head>
```
4. Tomcat 处理 GET 请求的字符集设置
前台网页的 GET 请求以 UTF-8 来解析，pom.xml 里设置 uriEncoding
``` xml
<!-- Web Server Tomcat -->
<plugin>
 <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>${tomcat.version}</version>
    <configuration>
        <path>/</path>
        <uriEncoding>UTF-8</uriEncoding> <!--处理 GET 的中文-->
    </configuration>
</plugin>
```
5. Tomcat 处理 POST 请求的字符集设置
前台网页的 POST 请求以 UTF-8 来解析，web.xml 加上字符集的 filter 处理 POST 的中文
``` xml
<!-- 处理 POST 的中文 -->
<filter>
    <filter-name>characterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
    <init-param>
        <param-name>forceEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>characterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

6. Freemarker 模版生成文件的字符集设置
在配置 Freemarker 的时候已经设置过了。
``` xml
<property name="contentType" value="text/html; charset=UTF-8"/>
```
7. AJAX 返回有乱码?
 * 由处理 @ResponseBody 返回字符串的 MessageConverter 的编码设置造成的，配置 spring-mvc.xml 中的 MessageConverter（去掉mvc:annotation-driven/）
 ``` xml
    <!-- 默认的注解映射支持 -->
    <!--<mvc:annotation-driven/>-->
	<mvc:annotation-driven>
   <mvc:message-converters register-defaults="true">
       <bean class="org.springframework.http.converter.StringHttpMessageConverter">
           <constructor-arg value="UTF-8" />
       </bean>
   </mvc:message-converters>
</mvc:annotation-driven>
 ```
8. 数据库的字符集设置
创建数据库的时候，选择 Encoding 为 UTF-8
9. 配置 JDBC 连接数据库的字符集为 UTF-8
``` xml
jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8
```
10. 使用 UTF-8 启动 Web Server(Tomcat)
	* Unix Like 配置 Tomcat
		在 Tomcat 启动文件 catalina.sh 中加一个 -Dfile.encoding=UTF-8
		改好后是这样
		```
			JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=UTF-8"
		```
	* Windows 配置 Tomcat
		在 Tomcat 启动文件 catalina.bat 中加一个 -Dfile.encoding=UTF-8
		修改好后是这样
		```
		set "JAVA_OPTS=%JAVA_OPTS% -Dfile.encoding=UTF-8"
		```