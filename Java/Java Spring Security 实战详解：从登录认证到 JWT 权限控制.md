### 简介

`Spring Security` 是 Spring 体系里专门处理安全问题的框架。

它主要解决三件事：

```text
登录认证：账号密码对不对
访问授权：这个账号能不能访问这个接口
安全防护：CSRF、Session 固定攻击、密码加密等
```

一句话概括：

```text
Spring Security 不是只做登录页，它是一套围绕请求过滤器展开的安全体系。
```

在 Spring Boot 项目里，引入 `spring-boot-starter-security` 后，所有接口默认都需要登录。这就是常说的“默认安全”。

实际项目里，最常见的用法有两类：

* 后台管理系统：表单登录、Session、角色权限
* 前后端分离接口：JSON 登录、JWT、无状态认证

这篇文章按 Spring Boot 3 和 Spring Security 6 的写法展开，不使用旧版 `WebSecurityConfigurerAdapter`。

### 认证和授权

Spring Security 最核心的两个词是 `Authentication` 和 `Authorization`。

### Authentication

`Authentication` 表示认证。

它回答的是：

```text
当前请求是谁发起的。
```

常见认证方式有：

* 用户名密码登录
* 手机号验证码登录
* JWT Token
* OAuth2 登录
* LDAP 登录

账号密码登录时，流程大概是：

```text
用户提交账号密码
  |
  v
Spring Security 校验账号是否存在
  |
  v
校验密码是否匹配
  |
  v
认证成功后生成 Authentication
  |
  v
放入 SecurityContext
```

认证成功后，业务代码就能拿到当前登录用户。

### Authorization

`Authorization` 表示授权。

它回答的是：

```text
当前用户能不能做这件事。
```

比如：

```text
GET /api/profile       登录用户可以访问
GET /api/admin/users   ADMIN 角色可以访问
DELETE /api/users/1    user:delete 权限可以访问
```

认证解决“是谁”，授权解决“能干什么”。

### 请求是怎么经过 Spring Security 的

Spring Security 的核心不是 Controller，也不是拦截器，而是 Servlet `Filter`。

一次请求进入应用后，大致会走这个链路：

```text
HTTP 请求
  |
  v
Servlet FilterChain
  |
  v
FilterChainProxy
  |
  v
SecurityFilterChain
  |
  v
认证过滤器
  |
  v
授权过滤器
  |
  v
Controller
```

几个核心对象可以这样理解：

| 对象 | 作用 |
| --- | --- |
| `SecurityFilterChain` | 配置哪些请求放行、哪些请求需要登录、哪些请求需要权限 |
| `SecurityContextHolder` | 保存当前线程里的登录信息 |
| `SecurityContext` | 安全上下文，里面放着 `Authentication` |
| `Authentication` | 当前认证对象，包含用户名、权限、是否认证成功 |
| `UserDetailsService` | 根据用户名加载用户信息 |
| `UserDetails` | Spring Security 认识的用户对象 |
| `GrantedAuthority` | 权限标识，比如 `ROLE_ADMIN`、`user:delete` |
| `PasswordEncoder` | 密码加密和密码匹配 |

### Maven 依赖

下面用 Spring Boot 3 做示例。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

如果只做 Session 登录演示，`jjwt` 这三个依赖可以先不加。

### 第一个 Demo：内存用户登录

先看最小可运行版本。

这个版本不连接数据库，直接在内存里放两个用户：

| 用户名 | 密码 | 角色 | 权限 |
| --- | --- | --- | --- |
| `admin` | `123456` | `ADMIN` | `user:list`、`user:delete` |
| `tom` | `123456` | `USER` | `user:list` |

配置类：

```java
package com.example.securitydemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .formLogin(Customizer.withDefaults())
                .logout(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {
        UserDetails admin = User.withUsername("admin")
                .password(passwordEncoder.encode("123456"))
                .roles("ADMIN")
                .authorities("user:list", "user:delete", "ROLE_ADMIN")
                .build();

        UserDetails tom = User.withUsername("tom")
                .password(passwordEncoder.encode("123456"))
                .roles("USER")
                .authorities("user:list", "ROLE_USER")
                .build();

        return new InMemoryUserDetailsManager(admin, tom);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

测试接口：

```java
package com.example.securitydemo.controller;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api")
public class DemoController {

    @GetMapping("/public")
    public Map<String, Object> publicApi() {
        return Map.of("message", "公开接口，不需要登录");
    }

    @GetMapping("/profile")
    public Map<String, Object> profile(Authentication authentication) {
        return Map.of(
                "username", authentication.getName(),
                "authorities", authentication.getAuthorities()
        );
    }

    @GetMapping("/admin/users")
    public Map<String, Object> adminUsers() {
        return Map.of("message", "只有 ADMIN 角色可以访问");
    }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasAuthority('user:delete')")
    public Map<String, Object> deleteUser(@PathVariable Long id) {
        return Map.of("message", "删除成功", "id", id);
    }
}
```

访问效果：

```text
GET /api/public
不登录也能访问

GET /api/profile
未登录会跳转到默认登录页

GET /api/admin/users
admin 可以访问，tom 不能访问

DELETE /api/users/1
拥有 user:delete 权限才可以访问
```

这个 demo 已经包含了 Spring Security 的基本骨架：

```text
SecurityFilterChain 配访问规则
UserDetailsService 提供用户
PasswordEncoder 处理密码
Authentication 表示当前登录用户
@PreAuthorize 做方法级权限控制
```

### Role 和 Authority 的区别

`Role` 和 `Authority` 很容易混。

Spring Security 底层真正判断的是 `Authority`。

`Role` 只是带固定前缀的特殊 `Authority`。

```java
.roles("ADMIN")
```

等价于添加：

```text
ROLE_ADMIN
```

所以：

```java
.requestMatchers("/api/admin/**").hasRole("ADMIN")
```

实际检查的是：

```text
ROLE_ADMIN
```

而下面这种写法检查的是普通权限：

```java
@PreAuthorize("hasAuthority('user:delete')")
```

它不会自动补 `ROLE_` 前缀。

常见约定：

```text
角色：ROLE_ADMIN、ROLE_USER
权限：user:list、user:add、user:delete
```

### 密码为什么不能明文存储

数据库里不能存：

```text
123456
```

应该存哈希后的结果：

```text
{bcrypt}$2a$10$......
```

推荐使用：

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

它返回的是 `DelegatingPasswordEncoder`。

密码格式通常长这样：

```text
{bcrypt}加密后的密码
```

前面的 `{bcrypt}` 表示使用哪种算法校验。后面算法升级时，可以兼容老密码。

生成密码可以写一个简单测试：

```java
PasswordEncoder encoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
String rawPassword = "123456";
String encodedPassword = encoder.encode(rawPassword);

System.out.println(encodedPassword);
System.out.println(encoder.matches(rawPassword, encodedPassword));
```

输出类似：

```text
{bcrypt}$2a$10$wbc0z12Yy0J7LQXMtWpm2uRld5qFf2qa7xM3k5NfR3Fkl.PzT4Bcq
true
```

每次生成的密文不一样，这是正常现象。`BCrypt` 会加入随机盐值。

### 数据库用户 Demo

内存用户适合学习，不适合真实项目。

真实项目通常从数据库加载用户、角色、权限。

下面用 JPA 和 H2 演示。换成 MySQL 时，实体和 Repository 基本不用变。

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:security_demo
    driver-class-name: org.h2.Driver
    username: sa
    password:

  h2:
    console:
      enabled: true

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true

app:
  jwt:
    secret: "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
    expire-minutes: 120
```

### 用户实体

```java
package com.example.securitydemo.entity;

import jakarta.persistence.Column;
import jakarta.persistence.ElementCollection;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "sys_user")
public class SysUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private Boolean enabled = true;

    @ElementCollection(fetch = FetchType.EAGER)
    private Set<String> roles = new HashSet<>();

    @ElementCollection(fetch = FetchType.EAGER)
    private Set<String> permissions = new HashSet<>();

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public Boolean getEnabled() {
        return enabled;
    }

    public void setEnabled(Boolean enabled) {
        this.enabled = enabled;
    }

    public Set<String> getRoles() {
        return roles;
    }

    public void setRoles(Set<String> roles) {
        this.roles = roles;
    }

    public Set<String> getPermissions() {
        return permissions;
    }

    public void setPermissions(Set<String> permissions) {
        this.permissions = permissions;
    }
}
```

### Repository

```java
package com.example.securitydemo.repository;

import com.example.securitydemo.entity.SysUser;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface SysUserRepository extends JpaRepository<SysUser, Long> {

    Optional<SysUser> findByUsername(String username);
}
```

### 初始化两条用户数据

```java
package com.example.securitydemo.config;

import com.example.securitydemo.entity.SysUser;
import com.example.securitydemo.repository.SysUserRepository;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Set;

@Configuration
public class DataInitConfig {

    @Bean
    public CommandLineRunner initUsers(SysUserRepository userRepository, PasswordEncoder passwordEncoder) {
        return args -> {
            if (userRepository.findByUsername("admin").isEmpty()) {
                SysUser admin = new SysUser();
                admin.setUsername("admin");
                admin.setPassword(passwordEncoder.encode("123456"));
                admin.setRoles(Set.of("ADMIN"));
                admin.setPermissions(Set.of("user:list", "user:add", "user:delete"));
                userRepository.save(admin);
            }

            if (userRepository.findByUsername("tom").isEmpty()) {
                SysUser tom = new SysUser();
                tom.setUsername("tom");
                tom.setPassword(passwordEncoder.encode("123456"));
                tom.setRoles(Set.of("USER"));
                tom.setPermissions(Set.of("user:list"));
                userRepository.save(tom);
            }
        };
    }
}
```

### 实现 UserDetailsService

Spring Security 不直接认识业务里的 `SysUser`。

它认识的是 `UserDetails`。

所以需要把数据库用户转换成 Spring Security 的用户对象。

```java
package com.example.securitydemo.security;

import com.example.securitydemo.entity.SysUser;
import com.example.securitydemo.repository.SysUserRepository;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class DbUserDetailsService implements UserDetailsService {

    private final SysUserRepository userRepository;

    public DbUserDetailsService(SysUserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SysUser user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("用户不存在"));

        List<GrantedAuthority> authorities = new ArrayList<>();

        for (String role : user.getRoles()) {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
        }

        for (String permission : user.getPermissions()) {
            authorities.add(new SimpleGrantedAuthority(permission));
        }

        return User.withUsername(user.getUsername())
                .password(user.getPassword())
                .disabled(!user.getEnabled())
                .authorities(authorities)
                .build();
    }
}
```

这里的关键点：

```text
数据库角色 ADMIN -> ROLE_ADMIN
数据库权限 user:delete -> user:delete
```

`hasRole("ADMIN")` 检查 `ROLE_ADMIN`。

`hasAuthority("user:delete")` 检查 `user:delete`。

### 前后端分离为什么常用 JWT

传统表单登录依赖 Session。

流程是：

```text
登录成功
  |
  v
服务端保存 Session
  |
  v
浏览器保存 JSESSIONID
  |
  v
后续请求带 Cookie
```

前后端分离、移动端、小程序、微服务场景里，经常选择 JWT。

流程变成：

```text
登录成功
  |
  v
服务端签发 JWT
  |
  v
前端保存 Token
  |
  v
后续请求放到 Authorization 请求头
  |
  v
服务端解析 Token 并恢复登录状态
```

请求头格式：

```text
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

JWT 的特点：

* 服务端不需要保存登录 Session
* Token 被篡改后签名校验会失败
* Token 过期后需要重新登录或刷新
* 已签发 Token 在过期前通常仍然有效，强制下线需要额外做黑名单或版本号

### JWT 工具类

```java
package com.example.securitydemo.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Date;
import java.util.List;

@Component
public class JwtService {

    private final SecretKey secretKey;
    private final long expireMinutes;

    public JwtService(
            @Value("${app.jwt.secret}") String secret,
            @Value("${app.jwt.expire-minutes}") long expireMinutes
    ) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.expireMinutes = expireMinutes;
    }

    public String generateToken(UserDetails userDetails) {
        Instant now = Instant.now();
        List<String> authorities = userDetails.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        return Jwts.builder()
                .subject(userDetails.getUsername())
                .claim("authorities", authorities)
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plusSeconds(expireMinutes * 60)))
                .signWith(secretKey)
                .compact();
    }

    public String getUsername(String token) {
        return parseClaims(token).getSubject();
    }

    public boolean isValid(String token, UserDetails userDetails) {
        String username = getUsername(token);
        Date expiration = parseClaims(token).getExpiration();
        return username.equals(userDetails.getUsername()) && expiration.after(new Date());
    }

    private Claims parseClaims(String token) {
        return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

`app.jwt.secret` 不能太短。

`HS256` 至少需要 256 bit 密钥，换算下来至少 32 字节。生产环境不要把密钥硬编码在代码里，通常放到环境变量、配置中心或密钥管理系统。

### 登录接口

先定义请求和响应对象。

```java
package com.example.securitydemo.dto;

public record LoginRequest(String username, String password) {
}
```

```java
package com.example.securitydemo.dto;

public record LoginResponse(String token, String tokenType) {
}
```

登录 Controller：

```java
package com.example.securitydemo.controller;

import com.example.securitydemo.dto.LoginRequest;
import com.example.securitydemo.dto.LoginResponse;
import com.example.securitydemo.security.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    public AuthController(AuthenticationManager authenticationManager, JwtService jwtService) {
        this.authenticationManager = authenticationManager;
        this.jwtService = jwtService;
    }

    @PostMapping("/login")
    public LoginResponse login(@RequestBody LoginRequest request) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password())
        );

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String token = jwtService.generateToken(userDetails);

        return new LoginResponse(token, "Bearer");
    }
}
```

这里没有手写密码校验。

这行代码会触发 Spring Security 的认证流程：

```java
authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(username, password)
);
```

底层会调用 `UserDetailsService` 查用户，再用 `PasswordEncoder` 校验密码。

### JWT 认证过滤器

登录接口只负责签发 Token。

后续请求还需要一个过滤器，从请求头里取 Token，校验后放入 `SecurityContextHolder`。

```java
package com.example.securitydemo.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        String token = resolveToken(request);

        try {
            if (token != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                String username = jwtService.getUsername(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtService.isValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authentication =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails,
                                    null,
                                    userDetails.getAuthorities()
                            );

                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        } catch (Exception ex) {
            SecurityContextHolder.clearContext();
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String authorization = request.getHeader("Authorization");

        if (StringUtils.hasText(authorization) && authorization.startsWith("Bearer ")) {
            return authorization.substring(7);
        }

        return null;
    }
}
```

`OncePerRequestFilter` 表示一次请求只执行一次。

核心动作只有三个：

```text
取出 Authorization 请求头
校验 JWT 并加载用户
把认证信息放入 SecurityContextHolder
```

### JWT 版 SecurityConfig

前后端分离接口通常不使用默认登录页，也不使用 Session。

配置如下：

```java
package com.example.securitydemo.config;

import com.example.securitydemo.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                )
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/auth/login", "/api/public").permitAll()
                        .requestMatchers("/h2-console/**").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .headers(headers -> headers
                        .frameOptions(frameOptions -> frameOptions.sameOrigin())
                )
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

这里有几个重点：

| 配置 | 说明 |
| --- | --- |
| `csrf(AbstractHttpConfigurer::disable)` | JWT API 不靠 Cookie 自动带身份，通常关闭 CSRF |
| `SessionCreationPolicy.STATELESS` | 不创建服务端 Session |
| `requestMatchers("/api/auth/login").permitAll()` | 登录接口必须放行 |
| `addFilterBefore(...)` | JWT 过滤器放在用户名密码过滤器前面 |
| `@EnableMethodSecurity` | 开启 `@PreAuthorize` |

如果是传统表单页面，并且认证状态依赖 Cookie 和 Session，就不要随手关闭 CSRF。

### 统一 JSON 错误响应

前后端分离项目里，未登录和无权限不适合返回 HTML。

可以自定义两个处理器。

未登录处理：

```java
package com.example.securitydemo.security;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class JsonAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException
    ) throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("""
                {"code":401,"message":"未登录或登录已过期"}
                """);
    }
}
```

无权限处理：

```java
package com.example.securitydemo.security;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class JsonAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException
    ) throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("""
                {"code":403,"message":"权限不足"}
                """);
    }
}
```

加到配置里：

```java
.exceptionHandling(exception -> exception
        .authenticationEntryPoint(jsonAuthenticationEntryPoint)
        .accessDeniedHandler(jsonAccessDeniedHandler)
)
```

完整构造器示例：

```java
private final JwtAuthenticationFilter jwtAuthenticationFilter;
private final JsonAuthenticationEntryPoint jsonAuthenticationEntryPoint;
private final JsonAccessDeniedHandler jsonAccessDeniedHandler;

public SecurityConfig(
        JwtAuthenticationFilter jwtAuthenticationFilter,
        JsonAuthenticationEntryPoint jsonAuthenticationEntryPoint,
        JsonAccessDeniedHandler jsonAccessDeniedHandler
) {
    this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    this.jsonAuthenticationEntryPoint = jsonAuthenticationEntryPoint;
    this.jsonAccessDeniedHandler = jsonAccessDeniedHandler;
}
```

### 测试 JWT 接口

登录：

```shell
curl -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"123456"}'
```

返回：

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer"
}
```

访问普通登录接口：

```shell
curl http://localhost:8080/api/profile \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...'
```

访问管理员接口：

```shell
curl http://localhost:8080/api/admin/users \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...'
```

删除用户：

```shell
curl -X DELETE http://localhost:8080/api/users/1 \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...'
```

`admin` 可以删除。

`tom` 访问删除接口会返回 `403`。

### 方法级权限控制

URL 级权限适合粗粒度控制：

```java
.requestMatchers("/api/admin/**").hasRole("ADMIN")
```

方法级权限适合业务层控制：

```java
@PreAuthorize("hasAuthority('user:delete')")
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
```

也可以拿参数做判断：

```java
@PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
public UserDetail getUserDetail(Long userId) {
    return userRepository.findDetail(userId);
}
```

上面这类写法需要自定义登录用户对象，把 `id` 放到 `principal` 里。

普通项目里常见分层方式：

```text
SecurityFilterChain 控制大方向
@PreAuthorize 控制具体业务动作
```

比如：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    @PreAuthorize("hasAuthority('user:list')")
    public List<String> list() {
        return List.of("admin", "tom");
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('user:delete')")
    public String delete(@PathVariable Long id) {
        return "deleted " + id;
    }
}
```

### CSRF 到底该不该关

`CSRF` 是跨站请求伪造。

它主要针对这种场景：

```text
浏览器会自动携带 Cookie
服务端又靠 Cookie 判断登录状态
```

如果是传统后台页面：

```text
表单登录 + Session + Cookie
```

通常应该开启 CSRF。

如果是前后端分离 API：

```text
Authorization: Bearer token
```

浏览器不会自动给第三方网站带上这个请求头，很多项目会关闭 CSRF。

所以不是一句“前后端分离就一定关闭”这么简单。

更准确的判断是：

```text
登录凭证是否会被浏览器自动携带。
```

靠 Cookie 自动带身份，要认真处理 CSRF。

靠 Authorization 请求头带 Token，通常可以关闭 CSRF。

### Session 和 JWT 怎么选

两种方式没有绝对优劣。

| 方案 | 优点 | 缺点 | 常见场景 |
| --- | --- | --- | --- |
| Session | 服务端可控，踢人、续期、失效都直接 | 集群需要共享 Session 或粘性会话 | 后台管理系统、传统 Web |
| JWT | 无状态，适合跨端和微服务 | Token 签发后不容易立刻失效 | App、小程序、开放 API、前后端分离 |

如果系统需要强制下线、单端登录、后台封禁立即生效，JWT 还需要配合：

* Redis 黑名单
* 用户 tokenVersion
* 短过期时间
* Refresh Token

JWT 不是万能登录方案，只是无状态认证方案。

### 常见坑

### 访问接口一直 401

先检查登录接口是否放行：

```java
.requestMatchers("/api/auth/login").permitAll()
```

再检查 JWT 请求头是否正确：

```text
Authorization: Bearer token内容
```

`Bearer` 后面有一个空格。

### 明明有 ADMIN 还是 403

检查角色是否带了 `ROLE_`。

下面写法：

```java
hasRole("ADMIN")
```

实际检查：

```text
ROLE_ADMIN
```

数据库里如果存的是 `ADMIN`，转换成 `GrantedAuthority` 时需要补上：

```java
new SimpleGrantedAuthority("ROLE_" + role)
```

### @PreAuthorize 不生效

配置类需要开启：

```java
@EnableMethodSecurity
```

Spring Boot 默认不会自动打开方法级权限控制。

### 密码一直匹配失败

数据库密码必须是 `PasswordEncoder` 加密后的结果。

错误示例：

```text
123456
```

正确示例：

```text
{bcrypt}$2a$10$......
```

如果使用 `DelegatingPasswordEncoder`，密文前面通常会有 `{bcrypt}`。

### JWT 过滤器里异常直接打断请求

Token 过期、签名错误、格式错误都可能抛异常。

生产代码建议在过滤器里捕获 JWT 解析异常，然后交给统一异常响应处理，避免返回默认错误页。

示例：

```java
try {
    String username = jwtService.getUsername(token);
    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
    if (jwtService.isValid(token, userDetails)) {
        // 设置认证信息
    }
} catch (Exception ex) {
    SecurityContextHolder.clearContext();
}
```

### 总结

Spring Security 可以按这条线理解：

```text
请求先进入过滤器链
过滤器负责识别登录身份
认证成功后放入 SecurityContext
授权规则决定接口能不能访问
Controller 和 Service 只处理业务
```

常用配置记住这几个点就够用：

* `SecurityFilterChain` 配 URL 访问规则
* `UserDetailsService` 从数据库加载用户
* `PasswordEncoder` 负责密码加密和匹配
* `AuthenticationManager` 负责执行登录认证
* `SecurityContextHolder` 保存当前登录状态
* `@EnableMethodSecurity` 开启方法权限
* `@PreAuthorize` 做细粒度权限判断
* JWT 项目一般配置 `STATELESS`，并把自定义过滤器加到过滤器链里

真正落地时，不要只盯着“登录成功”。还要一起处理密码加密、权限模型、未登录响应、无权限响应、Token 过期、强制下线和接口分层。安全代码最怕散，规则集中、边界清楚，后面扩展角色和权限才不容易乱。
