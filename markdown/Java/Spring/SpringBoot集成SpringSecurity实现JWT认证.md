# SpringBoot集成SpringSecurity实现JWT认证

## jwt

### 简介

JWT是一种用于双方之间传递安全信息的简洁的、URL安全的表述性声明规范。JWT作为一个开放的标准（ RFC 7519 ），定义了一种简洁的，自包含的方法用于通信双方之间以Json对象的形式安全的传递信息。因为数字签名的存在，这些信息是可信的，JWT可以使用HMAC算法或者是RSA的公私秘钥对进行签名。
        
### 特点

* 简洁(Compact)
    可以通过URL，POST参数或者在HTTP header发送，因为数据量小，传输速度也很快。
    
* 自包含(Self-contained)
    负载中包含了所有用户所需要的信息，避免了多次查询数据库。
    
### java平台的使用
    
#### maven 支持
    ``` java
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt</artifactId>
        <version>0.7.0</version>
    </dependency>
    ```
        
#### 生成 token

``` java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.impl.crypto.MacProvider;
import java.security.Key;

// We need a signing key, so we'll create one just for this example. Usually
// the key would be read from your application configuration instead.
Key key = MacProvider.generateKey();

String compactJws = Jwts.builder()
    .setSubject("Joe")
    .signWith(SignatureAlgorithm.HS512, key)
    .compact();
```
            
#### 解析token
``` java
String compactJws = "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJKb2UifQ.yiV1GWDrQyCeoOswYTf_xvlgsnaVVYJM0mU6rkmRBf2T1MBl3Xh2kZii0Q9BdX5-G0j25Qv2WF4lA6jPl5GKuA";
try {
    Jwts.parser().setSigningKey(key).parseClaimsJws(compactJws);
    //OK, we can trust this JWT
} catch (SignatureException e) {
    //don't trust the JWT!
}
```
## spring-security 集成 jwt 方案

* 在spring-security原本的FilterChain中，
添加 jwt认证用的Filter ：JwtAuthenticationTokenFilter。
    
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121234622.png)

* 客户端请求认证流程
        
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121234649.png)
    
### 具体实现
    
* application.properties 配置

``` java
    ##============JSON Web Token========================================
jwt.header=Authorization
jwt.secret=mySecret
jwt.expiration=604800            jwt.route.authentication.path=auth
jwt.route.authentication.refresh=refresh
jwt.route.authentication.register="auth/register"
```
        
* 添加 JwtAuthenticationTokenFilter
      
``` java
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    private final Log logger = LogFactory.getLog(this.getClass());

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Value("${jwt.header}")
    private String tokenHeader;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        System.out.println("进来了 JwtAuthenticationTokenFilter");

        // 得到 请求头的 认证信息 authToken
        String authToken = request.getHeader(this.tokenHeader);

        // 解析 authToken 得到 用户名
        String username = jwtTokenUtil.getUsernameFromToken(authToken);

        System.out.println("checking authentication for user " + username);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {

            // 根据用户名从数据库查找用户信息
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            // 检验token是否有效，并检验其保存的用户信息是否正确
            if (jwtTokenUtil.validateToken(authToken, userDetails)) {
                // token 有效，为该请求装载 用户权限信息
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                logger.info("authenticated user " + username + ", setting security context");
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }
        System.out.println("出去了 JwtAuthenticationTokenFilter");
        chain.doFilter(request, response);
    }
}
```
      
* 将 JwtAuthenticationTokenFilter 添加到 FilterChain 中
      
``` java
@EnableWebSecurity
public class MultiHttpSecurityConfig {

    @Configuration
    public static class FormLoginWebSecurityConfigurerAdapter extends WebSecurityConfigurerAdapter {

        @Autowired
        private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;

        // 静态资源访问的 url
        private String[] staticFileUrl = {};
        // 不用认证就可访问的 url
        private String[] permitUrl = {};

        @Override
        public void configure(WebSecurity web) throws Exception {
            web.ignoring().antMatchers(staticFileUrl);
            web.ignoring().antMatchers(permitUrl);
        }

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.csrf().disable();

            // 访问url认证
            http
                    .authorizeRequests()
                    .antMatchers("/admin/**").hasAuthority(String.valueOf(AuthorityName.ROLE_ADMIN))
                    .anyRequest().authenticated();
            // 配置登陆信息
            http
                    .formLogin().loginPage("/login")
                    .defaultSuccessUrl("/goIndex")
                    .permitAll()
                    .and();
            // 配置退出登陆信息
            http
                    .logout()
                    .logoutSuccessUrl("/login")
                    .invalidateHttpSession(true)
                    .deleteCookies()
                    .and();

            http.addFilterBefore(jwtAuthenticationTokenFilter,UsernamePasswordAuthenticationFilter.class);

            http.httpBasic();
        }
    }
}
```
      
* 提供一个jwt认证服务：提供 jwt token 的 生成和更新 功能（其实就是一个controller）
      
``` java
@RestController
@RequestMapping("authentication")
public class AuthenticationRestController {

    private final Log logger = LogFactory.getLog(this.getClass());

    @Value("${jwt.header}")
    private String tokenHeader;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Autowired
    private UserDetailsService userDetailsService;

    @RequestMapping(value = "${jwt.route.authentication.path}", method = RequestMethod.POST)
    public ResponseEntity<?> createAuthenticationToken(@RequestBody JwtAuthenticationRequest authenticationRequest, Device device) throws AuthenticationException {
        System.out.println("进来了 createAuthenticationToken ");

        System.out.println("authenticationRequest : " + authenticationRequest.getPassword() + "::" + authenticationRequest.getUsername());

        // Perform the security
        final Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        authenticationRequest.getUsername(),
                        authenticationRequest.getPassword()
                )
        );
        SecurityContextHolder.getContext().setAuthentication(authentication);

        // Reload password post-security so we can generate token
        final UserDetails userDetails = userDetailsService.loadUserByUsername(authenticationRequest.getUsername());
        final String token = jwtTokenUtil.generateToken(userDetails, device);

        // Return the token
        return ResponseEntity.ok(new JwtAuthenticationResponse(token));
    }
}
```
      
* 至此，spring-security 整合 jwt 认证 就已经完成了。
  
## 参考文档

[Spring Security Reference](http://docs.spring.io/spring-security/site/docs/5.0.0.BUILD-SNAPSHOT/reference/htmlsingle/)

[jjwt](https://github.com/jwtk/jjwt)

[使用JWT和Spring Security保护REST API](http://www.jianshu.com/p/6307c89fe3fa)