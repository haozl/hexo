title: Spring registration example
date: 2016-01-04 13:09:41
category: spring
---

#### Spring实现用户表单注册的简单例子,主要技术：
1. Spring Validator
2. Spring PasswordEncoder

### 定义UserDto模型:
```java
    @NotBlank
    @Size(min = 2, max = 32)
    private String username;

    @NotBlank
    @Size(min = 3, max = 32)
    private String password;

    @NotBlank
    @Size(min = 3, max = 32)
    private String passwordConfirmation;

    @NotBlank
    @Email
    private String email;

    @NotBlank
    private String captcha;
```
spring4 支持 [JSR-303/JSR-349](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/validation.html#validation-beanvalidation-overview) Bean Validation. 虽然这里我使用了`JSR-303 annotation`, 但这里我采用`Spring’s Validator interface`，因为Spring validator更适合复杂的验证逻辑，而且实现简单。( `JSR validation`也能自定义验证逻辑，[详情](http://stackoverflow.com/questions/11378320/using-jsr-303-validator-instead-of-spring-validator) )

用户注册验证实现如下：`RegistrationValidator.java`
```java
@Component
public class RegistrationValidator implements Validator {

    private static final String EMAIL_PATTERN =
            "^[_A-Za-z0-9-\\+]+(\\.[_A-Za-z0-9-]+)*@"
                    + "[A-Za-z0-9-]+(\\.[A-Za-z0-9]+)*(\\.[A-Za-z]{2,})$";

    @Override
    public boolean supports(Class<?> clazz) {
        return UserDto.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "username", "NotEmpty");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "password", "NotEmpty");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "passwordConfirmation", "NotEmpty");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "email", "NotEmpty");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "captcha", "NotEmpty");

        UserDto userDto = (UserDto) target;
        if (!StringUtils.equals(userDto.getPasswordConfirmation(), userDto.getPassword())) {
            errors.rejectValue("passwordConfirmation", "NotMatch.password");
        }

        Pattern pattern = Pattern.compile(EMAIL_PATTERN);
        if (!pattern.matcher(userDto.getEmail()).matches()) {
            errors.rejectValue("email", "Invalid");
        }
    }
}
```
`ValidationUtils.rejectIfEmptyOrWhitespace`验证字段是否空

`   if (!StringUtils.equals(userDto.getPasswordConfirmation(), userDto.getPassword())) {}`验证密码确认是否一致

最后是正则验证邮箱格式

### 注册用户逻辑：
`UserServiceImpl.java`
```java
    @Autowired
    UserDao userDao;

    @Autowired
    PasswordEncoder passwordEncoder;

    @Override
    public void registerNewUserAccount(UserDto userDto) throws EmailExistsException, UserExistsException {

        if (userDao.findByName(userDto.getUsername()) != null) {
            throw new UserExistsException("username already exists: " + userDto.getUsername());
        }
        if (userDao.findUserByEmail(userDto.getEmail()) != null) {
            throw new EmailExistsException("email already exists: " + userDto.getEmail());
        }

        final User user = new User();
        user.setUsername(userDto.getUsername().trim());
        String hashed = passwordEncoder.encode(userDto.getPassword().trim());
        user.setPassword(hashed);
        user.setEnabled(true);
        user.setEmail(userDto.getEmail().trim());
        userDao.save(user);

        userDao.addAuthority(user.getUsername(), "ROLE_USER");
    }
```
主要判断用户名和邮箱是否已存在，注意密码不能以明文方式或者简单的Hash处理后就存入数据库。Spring Security已经为我们提供安全有效的解决方法——`BCryptPasswordEncoder`。其算法是[bcrypt](https://www.wikiwand.com/en/Bcrypt)。
`bcrypt`算法每一次都会采用随机Salt，即使相同密码，每次得到结果都会不同：
```java
    String password = "123";
    for (int i = 0; i < 5; i++) {
        BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
        String hashed = encoder.encode(password);
        System.out.println(hashed);
        Assert.assertTrue(encoder.matches(password, hashed));
    }
```
上面会得到5个不同的结果：
```
$2a$10$p1xDtwHxpBopGVf6ixt0nuoO/Y8WNf9LnJA1L7gnTJp5cUQOvWpnK
$2a$10$15yF4r5J.9N0QSg8GhW1felHoWroCO/zk8EuA.8WGDBuAmVwiJUX2
$2a$10$nth9gZaElZX7doMQa.J7e.X357Mjk1t1s0ppwXPYSepsbIpO4sfnm
$2a$10$xxLzxXVN61DKHLNBd9lMpOiKyemTZJdWBEdVb8xJMg937VdR6l2be
$2a$10$rSp8aqPBdZtFiRGCdNEOMOSWV/Qttwva6GEJiLNMHWRazyY1y3QOO
```
`$2a$`是bcrypt算法识别前缀，`$10$`是加密强度，默认10，接着22个字符是`salt`，最后剩下的是`hash`.

在Spring中创建`PasswordEncoder` Bean:
```java
   @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
```
用户创建后先把密码`bcrypt`后再存入数据库：
```java
        String hashed = passwordEncoder.encode(userDto.getPassword().trim());
        user.setPassword(hashed);
```
>经过`PasswordEncoder`处理密码后，注意修改登陆验证，例如为Jdbc验证加上passwordEncoder：
>
```java
    auth.jdbcAuthentication().dataSource(dataSource)
            .passwordEncoder(passwordEncoder());
```
或者在自定义验证中验证密码：
```java
passwordEncoder.matches(password, hashed);
```

### 注册Controller
```java
    @RequestMapping(value = "/register", method = RequestMethod.GET)
    public String showRegister(Model model) {
        model.addAttribute("user", new UserDto());
        return "register";
    }

    @RequestMapping(value = "/register", method = RequestMethod.POST)
    public String registerAccount(@ModelAttribute("user") /*@Valid*/ UserDto user, Errors errors,
                                  Model model, HttpServletRequest request, RegistrationValidator validator) {

        validator.validate(user, errors);
        String captcha1 = user.getCaptcha();
        String captcha2 = getGeneratedKey(request);
        if (!StringUtils.equals(captcha1, captcha2)) {
            errors.rejectValue("captcha", "Invalid");
        }

        if (errors.hasErrors()) {
            return "register";
        }

        try {
            userService.registerNewUserAccount(user);

            // generate session if one doesn't exist
            request.getSession();
            UsernamePasswordAuthenticationToken token =
                    new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword());
            SecurityContextHolder.getContext().setAuthentication(token);

            return "redirect:/";

        } catch (EmailExistsException e) {
            errors.rejectValue("email", "exists");
        } catch (UserExistsException e) {
            errors.rejectValue("username", "exists");
        }
        return "register";
    }

    @RequestMapping(value = "/captcha", method = RequestMethod.GET)
    public void captcha(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        super.captcha(request, response);
    }
```
把上文的`RegistrationValidator`注入到`registerAccount`方法中, 进行表单验证:`validator.validate(user, errors);`

最后是验证码部分：(采用 [kaptcha](https://github.com/axet/kaptcha))
```java
   String captcha1 = user.getCaptcha();
    String captcha2 = getGeneratedKey(request);
    if (!StringUtils.equals(captcha1, captcha2)) {
        errors.rejectValue("captcha", "Invalid");
    }
```

### 注册Form:
```html
        <form:form class="w3-container" modelAttribute="user" method="POST" enctype="utf8">
            <p>
                <form:label path="username" cssClass="w3-label"
                            cssErrorClass="">Username: </form:label>
                <form:input path="username" cssClass="w3-input"/>
                <form:errors path="username" cssClass="error"/>
            </p>
            <p>
                <form:label path="password" cssClass="w3-label"
                            cssErrorClass="">Password: </form:label>
                <form:password path="password" cssClass="w3-input"/>
                <form:errors path="password" cssClass="error"/>
            </p>
            <p>
                <form:label path="passwordConfirmation" cssClass="w3-label"
                            cssErrorClass="">Password Confirm: </form:label>
                <form:password path="passwordConfirmation" cssClass="w3-input"/>
                <form:errors path="passwordConfirmation" cssClass="error"/>
            </p>

            <p>
                <form:label path="email" cssClass="w3-label" cssErrorClass="">Email: </form:label>
                <form:input path="email" cssClass="w3-input"/>
                <form:errors path="email" cssClass="error"/>
            </p>

            <div class="w3-row">
                <p class="w3-half">
                    <form:label path="captcha" cssClass="w3-label" cssErrorClass="">Captcha: </form:label>
                    <form:input path="captcha" cssClass="w3-input"/>
                    <form:errors path="captcha" cssClass="error"/>
                </p>

                <p class="w3-half">
                    <img id="captcha_img" src="${captchaUrl}" alt="captcha" style="padding: 10px 30px">
                </p>
            </div>


            <a href="${loginUrl}" class="w3-right"><spring:message
                    code="form.login"></spring:message></a>
            <p>
                <button class="w3-btn w3-teal"><spring:message code="label.form.submit"></spring:message></button>
            </p>

        </form:form>
```        

### 简单测试注册功能
```java
 @Test
    public void registerValidUser() throws Exception {
        UserDto user = new UserDto();
        String username = "user123";
        String password = "123456";
        String email = "user@user.com";
        String captcha = "invalid";
        user.setUsername(username);
        user.setPassword(password);
        user.setPasswordConfirmation(password);
        user.setEmail(email);
        user.setCaptcha(captcha);
        
        //先请求验证码，并保存session
        HttpSession session = mockMvc.perform(get(CAPTCHA_URL))
                .andExpect(status().isOk())
                .andReturn()
                .getRequest()
                .getSession();

        String captcha = (String) session.getAttribute("KAPTCHA_SESSION_KEY");
        Assert.assertNotNull(captcha);
        Assert.assertNotNull(session);
        user.setCaptcha(captcha); //提取session中的验证码，准备提交

        mockMvc.perform(post(REGISTER_URL)
                .session((MockHttpSession) session) //带上验证码的session
                .locale(Locale.ENGLISH)
                .param("username", user.getUsername())
                .param("password", user.getPassword())
                .param("passwordConfirmation", user.getPasswordConfirmation())
                .param("email", user.getEmail())
                .param("captcha", user.getCaptcha())
        )
                .andDo(print())
                .andExpect(status().is3xxRedirection())
                .andExpect(model().attributeExists("user"))
                .andExpect(model().attribute("user", hasProperty("username", is(user.getUsername()))))
                .andExpect(model().attribute("user", hasProperty("password", is(user.getPassword()))))
                .andExpect(model().attribute("user", hasProperty("passwordConfirmation", is(user.getPasswordConfirmation()))))
                .andExpect(model().attribute("user", hasProperty("email", is(user.getEmail()))))
                .andExpect(model().attribute("user", hasProperty("captcha", is(user.getCaptcha()))))
                .andExpect(model().attributeHasNoErrors("user"))
        ;

        User saved = userService.findByName(user.getUsername());
        Assert.assertNotNull(saved);
        Assert.assertEquals(user.getUsername(), saved.getUsername());
        Assert.assertTrue(passwordEncoder.matches(user.getPassword(), saved.getPassword()));

    }
```

### [Source on Github](https://github.com/haozl/spring-samples/tree/master/register)
