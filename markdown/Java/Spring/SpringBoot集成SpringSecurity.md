# SpringBoot集成SpringSecurity

    
## Maven 依赖
        
    ``` java
            <!-- spring security 认证授权 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    ```
    
## 配置文件
        
``` java
@EnableWebSecurity
public class MultiHttpSecurityConfig {

    @Configuration
    public static class FormLoginWebSecurityConfigurerAdapter extends WebSecurityConfigurerAdapter {

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
            http.httpBasic();
        }
    }
}
```
        
## 有效性校验和授权

使用数据库的用户信息，进行对登陆的form提交的信息，进行验证。验证成功后为该用户配置相应的权限。
        
``` java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUserName(username);

        if (user == null) {
            throw new UsernameNotFoundException(String.format("No user found with username '%s'.", username));
        } else {
            return JwtUserFactory.create(user);
        }
    }
}
```
        
## 注意
            
* 需要实现下列接口及方法

```
import org.springframework.security.core.userdetails.UserDetailsService;
```

```
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```
        
* 至此，springboot 整合 springsecurity 已经完成，不过，对于权限认证，使用的是 form 表单提交登陆的方式。