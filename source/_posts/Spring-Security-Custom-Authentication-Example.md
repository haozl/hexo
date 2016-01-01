
title: Spring Security 自定义Authentication例子
date: 2016-01-01 21:09:41
category: spring
---

### 有两种方式可以实现自定义认证：
1. 自定义实现`AuthenticationProvider`:
```java
    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(customAuthenticationProvider); //custom AuthenticationProvider
    }
```

2. 自定义实现`UserDetailsService`:
```java
    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
         auth.userDetailsService(authenticationService); //UserDetailsService
    }
```
### 两种方式区别主要在于：当你的程序不能提供认证所需的密码时候，必须使用AuthenticationProvider认证，例如：CAS Authentication. 相反，你的程序能提供用户名和密码时候，一般采用UserDetailsService.

## 大概实现如下：

### 1. SecurityConfig.java
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    DataSource dataSource;

    @Autowired
    AuthenticationService authenticationService;

    @Autowired
    CustomAuthenticationProvider customAuthenticationProvider;

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {

//        auth.userDetailsService(authenticationService); //UserDetailsService

        auth.authenticationProvider(customAuthenticationProvider); //custom AuthenticationProvider

    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/resources/**", "/join").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login").permitAll()
                .and().logout().permitAll()
                .and().rememberMe();
    }
}
```

### 2. 实现AuthenticationProvider： CustomAuthenticationProvider.java
```java
@Service
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    UserDAO userDAO;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = (String) authentication.getPrincipal();
        String password = (String) authentication.getCredentials();

        UserInfo userInfo = userDAO.findByName(username);
        if (userInfo == null)
            throw new BadCredentialsException(username + "not exists");

        if (!userInfo.getPassword().equals(password))
            throw new BadCredentialsException("Bad Credentials");

        Authentication token = new UsernamePasswordAuthenticationToken(username, null, userInfo.getAuthorities());
        return token;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

### 3. 实现UserDetailsService： AuthenticationService
```java
@Service
public class AuthenticationService implements UserDetailsService {

    @Autowired
    UserDAO userDAO;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        UserInfo userInfo = userDAO.findByName(username);
        if (userInfo == null)
            throw new UsernameNotFoundException(username + " not exists!");
        User user = new User(userInfo.getUsername(), userInfo.getPassword(), userInfo.getAuthorities());
        return user;
    }
}
```

### 4. Database Schema
```sql
DROP TABLE users IF EXISTS;
CREATE TABLE users (
  id       INTEGER PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(32) UNIQUE NOT NULL,
  password VARCHAR(64)        NOT NULL,
  enabled  TINYINT            NOT NULL DEFAULT 1,
  email    VARCHAR(32) UNIQUE
);

DROP TABLE user_roles IF EXISTS;
CREATE TABLE user_roles (
  id      INTEGER PRIMARY KEY AUTO_INCREMENT,
  user_id INTEGER     NOT NULL,
  role    VARCHAR(48) NOT NULL,
  UNIQUE KEY role_user( role, user_id)
);




```

### 5. UserDAOImpl.java
```java
public class UserDAOImpl implements UserDAO {

    @Autowired
    JdbcTemplate jdbcTemplate;

    @Override
    public UserInfo findByName(String username) {
        UserInfo user;

        try {
            user = jdbcTemplate.queryForObject("select id, username, password from users where enabled=1 and username=?",
                    new Object[]{username}, new RowMapper<UserInfo>() {
                        @Override
                        public UserInfo mapRow(ResultSet rs, int rowNum) throws SQLException {
                            UserInfo user = new UserInfo();
                            user.setId(rs.getLong("id"));
                            user.setUsername(rs.getString("username"));
                            user.setPassword(rs.getString("password"));
                            return user;
                        }
                    });
        } catch (EmptyResultDataAccessException e) {
            return null;
        }

        List<String> roles = jdbcTemplate.queryForList("select role from user_roles where user_id=?", new Object[]{user.getId()}, String.class);
        user.setAuthorities(roles);

        return user;
    }
}
```

### [Source on Github](https://github.com/haozl/spring-samples/tree/master/authentication)
###  Run the application
```shell
$ git clone [this repository]
$ mvn jetty:run
```
Visit `http://localhost:8080`

###  Import the sample project into Eclipse IDE
1. ```$ mvn eclipse:eclipse```
2. File->Import->Existing projects into workspace
