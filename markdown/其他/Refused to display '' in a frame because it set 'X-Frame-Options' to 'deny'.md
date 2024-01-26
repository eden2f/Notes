# Refused to display '' in a frame because it set 'X-Frame-Options' to 'deny'

## X-Frame-Options 响应头

* X-Frame-Options [HTTP](https://developer.mozilla.org/en/HTTP) 响应头是用来给浏览器指示允许一个页面可否在 ``` <frame><iframe> ``` 或者 ``` <object> ``` 中展现的标记。网站可以使用此功能，来确保自己网站的内容没有被嵌到别人的网站中去，也从而避免了点击劫持 (clickjacking) 的攻击。
* X-Frame-Options 有三个值:
  * DENY : 表示该页面不允许在 frame 中展示，即便是在相同域名的页面中嵌套也不允许。
  * SAMEORIGIN : 表示该页面可以在相同域名页面的 frame 中展示。
  * ALLOW-FROM *uri* : 表示该页面可以在指定来源的 frame 中展示。
* 换一句话说，如果设置为 DENY，不光在别人的网站 frame 嵌入时会无法加载，在同域名页面中同样会无法加载。另一方面，如果设置为 SAMEORIGIN，那么页面就可以在同域名页面的 frame 中嵌套。

## 解决方法

### Apache

配置 Apache 在所有页面上发送 X-Frame-Options 响应头，需要把下面这行添加到 'site' 的配置中:
``` java
    Header always append X-Frame-Options SAMEORIGIN
```

### Nginx

配置 nginx 发送 X-Frame-Options 响应头，把下面这行添加到 'http', 'server' 或者 'location' 的配置中:
``` java
    add_header X-Frame-Options SAMEORIGIN;
```

### IIS

配置 IIS 发送 X-Frame-Options 响应头，添加下面的配置到 Web.config 文件中:
``` java 
    <system.webServer>
    ...
        <httpProtocol>
        <customHeaders>
            <add name="X-Frame-Options" value="SAMEORIGIN" />
        </customHeaders>
        </httpProtocol>
        ...
    </system.webServer>
```
### SpringBoot项目

springboot 出现这种问题的话，应该是配置了权限后才会的吧。这里以springsecurity 作权限控制 为例，配置方法如下 
``` java
    (HttpSecurity) http.headers().frameOptions().sameOrigin()
```

### Servlet的Fillter
  ``` java
  public class MyFilter implements Filter { 

      public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {  
          HttpServletRequest request = (HttpServletRequest) req;    
          HttpServletResponse response = (HttpServletResponse) res;    
      response.setHeader("x-frame-options", "SAMEORIGIN");    
      chain.doFilter(request, response);  
      }    

      public void init(FilterConfig config) throws ServletException {  
      }  

      public void destroy() {  
      }  
  }  
  ```
