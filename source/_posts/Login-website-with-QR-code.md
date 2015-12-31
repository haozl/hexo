title: 微信扫描二维码登陆网站的简单实现( spring+stomp )
date: 2015-12-29 08:49:41
tags: [qr, spring, websocket, stomp]
category: spring
---

当你用手机扫描[微信网页版](https://wx.qq.com/)的二维并在手机上授权登陆后，网页就会自动完成登陆。很神奇对吧？同样，支付宝手机支付完成后，起网页也会提示支付成功。
其原理基本是手机授权成功后，服务器生成access token,并且发送到前端网页,然后网页利用token完成登陆
手机客户端授权后，怎样通知前端网页是关键所在。本文将采用[spring-messaging](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/websocket.html) + [STOMP](http://www.wikiwand.com/en/Streaming_Text_Oriented_Messaging_Protocol) 来实现


## 基本实现步骤

1.用户访问首页，与服务器建立长连接[SockJS](https://github.com/sockjs)
2.用户扫描二维码，并授权登陆
3.服务器生成 access token，发送到前端网页
4.网页利用token完成登陆
>为了方便演示，步骤2的手机扫描二维码改成用网页模拟授权

## 效果图
![](/images/7da67a21bdf36180bad3c151593ecbba10cf980c.gif)


## 使用技术
* Spring 4.2.4.RELEASE
* Spring Security 4.0.3.RELEASE
* Maven 3
* H2 Database

## 具体实现

### 1. 首页index
```html
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>微信网页版</title>
    <link rel="stylesheet" href="<c:url value="/resources/css/bootstrap.min.css" />">
    <script src="<c:url value="/resources/sockjs-0.3.4.js" />"></script>
    <script src="<c:url value="/resources/stomp.js" />"></script>
    <script src="<c:url value="/resources/jquery.min.js" />"></script>
</head>
<body>
<div class="container">
    <div class="row">
        <h2>扫描二维码登录微信</h2>
        <img src="<c:url value="/resources/img/qr.jpg" />" alt="">
    </div>
    <div class="row">
        <button id="sim" class="btn btn-default btn-lg" target="_blank">模拟手机登陆</button>
    </div>
</div>
<script>
    var stompClient = null;
    var destination = Math.random().toString(36).slice(2);
    function connect() {
        var socket = new SockJS('/weixin');
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function (frame) {
            console.log('Connected: ' + frame);
            stompClient.subscribe('/topic/auth/' + destination, function (resp) {
                var token = resp.body;
                window.location.href = '<c:url value="/weixin/auth-token?token=" />' + token;
            });
        });
    }

    function disconnect() {
        if (stompClient != null) {
            stompClient.disconnect();
        }
        console.log("Disconnected");
    }

    $(function () {
        $('#sim').click(function () {
            var settings = "width=640, height=280, top=220, left=820, scrollbars=no, location=no, directories=no, status=no, menubar=no, toolbar=no, resizable=no, dependent=no";
            var url = '<c:url value="/weixin/snapshot?destination=" />' + destination;
            win = window.open(url, 'Phone', settings);
            win.focus();
        });
        connect();
    });

</script>
</body>
</html>

```
注意`stompClient`建立连接后，订阅`/topic/auth/ + destination`。这里的`var destination = Math.random().toString(36).slice(2);`
是随机生成的地址，用于标示客户端地址，稍后会发送到服务器。服务器生成access token后会投递到/topic/auth/ + destination,我们的客户端即可接受


### 2. 配置spring WebSocketConfig
WebSocketConfig.java
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/weixin").withSockJS();
    }

}
```
enableSimpleBroker("/topic") 建立broker
registerStompEndpoints 注册"/weixin" endpoint


### 3. WeiXinController.java
```java
@Controller
@RequestMapping("/weixin")
public class WeiXinController {

    @Autowired
    SimpMessagingTemplate messagingTemplate;

    @Autowired
    JdbcTemplate jdbcTemplate;

    @RequestMapping(value = "/snapshot", method = RequestMethod.GET)
    public String snapshot(Model model, String destination) {
        model.addAttribute("destination", destination);
        return "snapshot";
    }

    /**
     * generate user access token
     *
     * @param username
     * @return
     */
    @RequestMapping(value = "/snapshot", method = RequestMethod.POST, produces = MediaType.TEXT_HTML_VALUE)
    public
    @ResponseBody
    String auth(String username, String destination) {

        String token = UUID.randomUUID().toString();
        jdbcTemplate.update("update users set token=? where username=?", token, username);

        messagingTemplate.convertAndSend("/topic/auth/" + destination, token);
        String ret = String.format("<h2>%s has been authenticated!<h2>", username);
        return ret;
    }

    /**
     * Authenticate user with given token
     *
     * @param token
     * @return
     */
    @RequestMapping(value = "/auth-token")
    public String authWithToken(String token) {
        User user = jdbcTemplate.queryForObject("select * from users where token=?",
                new Object[]{token}, new UserRowMapper());

        //destroy token
        jdbcTemplate.update("update users set token=null where token=?", user.getToken());

        //authenticate user programmatically
        UsernamePasswordAuthenticationToken authenticationToken =
                new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword());
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);

        return "redirect:/user";
    }

    private class UserRowMapper implements RowMapper<User> {


        @Override
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            User user = new User();
            user.setId(rs.getInt("id"));
            user.setUsername(rs.getString("username"));
            user.setPassword(rs.getString("password"));
            user.setEnabled(rs.getBoolean("enabled"));
            user.setToken(rs.getString("token"));
            return user;
        }
    }

    @ExceptionHandler(EmptyResultDataAccessException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    private
    @ResponseBody
    String notFound() {
        return "Access denied!";
    }
}

```
`String auth(String username, String destination)` 处理手机授权请求，生成随机access token存入数据库，通过`messagingTemplate`把token投递到客户端。
( 这里`messagingTemplate.convertAndSend("/topic/auth/" + destination, token`) 中的destination就是客户端随机生成的地址)


`String authWithToken(String token)` 处理access token登陆，授权对应用户，删除access token。最后重定向用户到`/user`


### 4. 客户端收到token
最后客户端收到token后， `window.location.href = '<c:url value="/weixin/auth-token?token=" />' + token;`跳转到`/auth-token`进行登陆认证

---
## 源码: https://github.com/haozl/qr-login

## Reference
* http://hayageek.com/web-whatsapp-qr-code/
* http://blog.self.li/post/14864315302/qr-login-howto
* https://www.grc.com/sqrl/sqrl.htm