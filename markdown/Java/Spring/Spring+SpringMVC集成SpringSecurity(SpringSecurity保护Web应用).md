# Spring+SpringMVC集成SpringSecurity(SpringSecurity保护Web应用)

### 示例环境

1. JDK 1.8
2. Tomcat 8
3. spring 4.0.0.RELEASE
4. spring-security 4.0.0.RELEASE
5. IntelliJ IDEA 2017
6. Gradle 4.6

### jar包依赖

``` nxml
group 'com.xxxx'
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

    // spring security
    compile group: 'org.springframework.security', name: 'spring-security-core', version: '4.0.0.RELEASE'
    compile group: 'org.springframework.security', name: 'spring-security-web', version: '4.0.0.RELEASE'
    compile group: 'org.springframework.security', name: 'spring-security-config', version: '4.0.0.RELEASE'
//    compile group: 'org.springframework.security.oauth', name: 'spring-security-oauth2'

    compile group: 'org.springframework', name: 'spring-jdbc', version: '4.0.0.RELEASE'
    compile 'mysql:mysql-connector-java:6.0.6'

    compile group: 'javax.el', name: 'javax.el-api', version: '2.2.4'
    compile group: 'javax.validation', name: 'validation-api', version: '1.1.0.Final'
    compile group: 'org.hibernate', name: 'hibernate-validator', version: '5.2.2.Final'


    compile group: 'javax.servlet', name: 'javax.servlet-api', version: '3.1.0'
    compile group: 'taglibs', name: 'standard', version: '1.1.2'
    compile group: 'jstl', name: 'jstl', version: '1.2'

    compile group: 'org.apache.tiles', name: 'tiles-servlet', version: '3.0.5'
    compile group: 'org.apache.tiles', name: 'tiles-jsp', version: '3.0.5'
    compile group: 'org.apache.tiles', name: 'tiles-core', version: '3.0.5'
    compile group: 'org.apache.tiles', name: 'tiles-api', version: '3.0.5'

    compile group: 'org.slf4j', name: 'slf4j-log4j12', version: '1.7.25'

    testCompile group: 'junit', name: 'junit', version: '4.12'
    testCompile group: 'org.mockito', name: 'mockito-all', version: '1.10.19'
}
```

### 启用spring-security

#### 过滤Web请求

Spring Security借助一系列Servlet Filter来提供各种安全性功能。DelegatingFilterProxy是一个特殊的Servlet Filter,它本身所做的工作并不多。只是将工作委托给一个javax.servlet.Filter实现类,这个实现类作为一个bean注册在Spring应用的上下文中。

DelegatingFilterProxy把Filter的处理逻辑委托给Spring应用上下文中所定义的一个代理FilterBean

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240216133452.png)

借助WebApplicationInitializer以Java的方式来配 置DelegatingFilterProxy: 创建一个扩展的新类

``` java
public class SecurityWebInitializer extends AbstractSecurityWebApplicationInitializer {
}
```

#### 编写简单的安全性配置

启用Web安全性功能的最简单配置

``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter{
}
```

默认的configure(HttpSecurity),配置如下所示

``` java
// @formatter:off
protected void configure(HttpSecurity http) throws Exception {
    logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

    http
        .authorizeRequests()
            .anyRequest().authenticated()
            .and()
        .formLogin().and()
        .httpBasic();
}
// @formatter:on
```
  
这个简单的默认配置指定了该如何保护HTTP请求,以及客户端认证用户的方案。通过调用authorizeRequests()和anyRequest().authenticated()就会要求所有进入应用的HTTP请求都要进行认证。它也配置Spring Security支持基于表单的登录以及HTTPBasic方式的认证。

同时,因为我们没有重载configure(AuthenticationManagerBuilder)方法,所以没有用户存储支撑认证过程。没有用户存储,实际上就等于没有用户。所以,在这里所有的请求都需要认证,但是没有人能够登录成功。

为了让Spring Security满足我们应用的需求,还需要再添加一点配置。具体来讲,我们需要:
1. 配置用户存储;
2. 指定哪些请求需要认证,哪些请求不需要认证,以及所需要的权限;

### 配置用户登录认证信息读取来源

#### 使用基于内存的用户存储
``` java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
  // 启用内存用户存储
  auth
    .inMemoryAuthentication()
    .withUser("user").password("user").roles("USER").and()
    .withUser("admin").password("admin").roles("USER", "ADMIN");
 }
```
对于调试和开发人员测试来讲,基于内存的用户存储是很有用的,但是对于生产级别的应用
来讲,这就不是最理想的可选方案了。为了用于生产环境,通常最好将用户数据保存在某种
类型的数据库之中。

#### 基于数据库表进行认证
``` java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 基于数据库表进行认证
    auth
            .jdbcAuthentication();
}
```

尽管默认的最少配置能够让一切运转起来,但是它对我们的数据库模式有一些要求。它预期存在某些存储用户数据的表。更具体来说,下面的代码片段来源于Spring Security内部,这块代码展现了当查找用户信息时所执行的SQL查询语句:

``` sql
select username, password, enabled
from users
where username = ?;

select username, authority
from authorities
where username = ?;

select g.id, g.group_name, ga.authority
from groups, a, group_members gm, group_authorities ga
where gm.username = ?
and g.id = ga.group_id
and g.id = gm.group_id;
```

如果你能够在数据库中定义和填充满足这些查询的表,那么基本上就不需要你再做什么额外的事情了。但是,也有可能你的数据库与上面所述并不一致,那么你就会希望在查询上有更多的控制权。如果是这样的话,我们可以按照如下的方式配置自己的查询:

``` java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 基于数据库表进行认证
    auth
            .jdbcAuthentication()
            .dataSource(dataSource)
                .usersByUsernameQuery(
                        "select username, password, true " +
                        "from Spitter " +
                        "where username=? "
                )
                .authoritiesByUsernameQuery(
                        "select username, 'ROLE_USER' " +
                        "from Spitter  " +
                        "where username=? "
                );
}
```

#### 设置密码转换

Spring Security的加密模块包括了三个这样的实现:BCryptPasswordEncoder、NoOpPasswordEncoder和StandardPasswordEncoder。

``` java
// 基于数据库表进行认证
auth
        .jdbcAuthentication()
        .dataSource(dataSource)
            .usersByUsernameQuery(
                    "select username, password, true " +
                    "from Spitter " +
                    "where username=? "
            )
            .authoritiesByUsernameQuery(
                    "select username, 'ROLE_USER' " +
                    "from Spitter  " +
                    "where username=? "
            )
        //密码如果是需要转码，使用该方法配置解码器
        .passwordEncoder(new StandardPasswordEncoder("asdfjlsa"));
```

#### 配置自定义的用户服务
实现loadUserByUsername()方法,根据给定的用户名来查找用户。loadUserByUsername()方法会返回代表给定用户的UserDetails对象。如下的程序清单展现了一个UserDetailsService的实现,它会从数据库中查找用户信息。

``` java
@Service
public class SpitterServiceImpl implements UserDetailsService{

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 根据 username 查询数据库 得到 用户信息
        // ..... 
        Collection<SimpleGrantedAuthority> authorities = new ArrayList<>();
        SimpleGrantedAuthority authority = new SimpleGrantedAuthority("ROLE");
        authorities.add(authority);
        return new User("customUser", "customUser", authorities);
    }
}
```
使用SpitterUserService来认证用户,我们可以通过userDetailsService()方法将其设置到安全配置中：
``` java
@Autowired
private SpitterServiceImpl spitterService;

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 配置自定义的用户服务
    auth
            .userDetailsService(spitterService);
}
```

### 配置拦截请求规则
1. 使用Spring表达式进行安全保护
2. 强制通道的安全性
3. 防止跨站请求伪造
4. 认证用户
5. 启用HTTP Basic认证
6. 启用Remember-me功能
7. 清除登录信息

对每个请求进行细粒度安全性控制的关键在于重载configure(HttpSecurity)方法。下的代码片段展现了重载的configure(HttpSecurity)方法,它为不同的URL路径有选择地应用安全性:

``` java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .authorizeRequests()
                .anyRequest()
                .authenticated()  // 允许认证过的用户访问
                .and()
            .requiresChannel() // 强制使用 https 安全通道
                // 使用Spring表达式进行安全保护
                .antMatchers(HttpMethod.GET, "/spitter/register").requiresSecure()
                .and()
            .formLogin()
                .permitAll()  // 无条件允许访问
                .and()
            .httpBasic()
                .realmName("Spittr")
                .and()
            .rememberMe() // 如果用户是通过Remember-me功能认证的,就允许访问
                // 有效期
                .tokenValiditySeconds(2419200)
                // md5哈希 私钥
                .key("spittrKey")
                .and()
            .logout()
                .logoutSuccessUrl("/")
                .logoutUrl("/logout")
                .and()
            .csrf()
                // 禁用 csrf 防护功能
                .disable();
}
```

### 验证

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240216133907.png)




  
  
