---
layout:     post
title:      "微信小程序 + WebSocket 的java后端实现"
subtitle:   ""
date:       2019-03-14 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
    - Nginx
---


https://blog.csdn.net/qq_28988969/article/details/76057789
[小程序开发 - websocket](https://www.xncoding.com/2017/12/15/weixin/ma-websocket.html)
https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-websocket

## Nginx 配置WebSocket

小程序只允许https的WebSocket，并且需要域名，nginx需要配置好SSL证书


### 命令行
```shell
cd /usr/local/nginx
vim nginx.conf
./nginx -s reload
```
### nginx.conf 内容

```shell
http {
    
    # 动态升级WebSocket
    map $http_upgrade $connection_upgrade {
         default upgrade;
         ''      close;
    }

    server {
        listen 80;
        listen 443 ssl;
        server_name  # xxx.xxx.xxx;

        ssl on;
        ssl_certificate # ./nginx_conf/xxx.xxx.crt;
        ssl_certificate_key # ./nginx_conf/xxx.xxx.key;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers HIGH:!aNULL:!MD5:!DH;
        ssl_prefer_server_ciphers on;


        #WebSocket 配置
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        #prevents 502 bad gateway error
        proxy_buffers 8 32k;
        proxy_buffer_size 64k;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

}
```


## Java 配置


```java
/**
 * WebSocket 配置
 *
 * @author by SunYuXing on 2019-01-15.
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    private final HeartSocketHandler handler;

    @Autowired
    public WebSocketConfig(HeartSocketHandler handler) {
        this.handler = handler;
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        /*
        * 注册Hanlder到配置的path, 最终访问网址为 baseUrl/contextpath/path
        * 添加拦截器, 处理认证或其他切面操作
        * 允许跨域
        */
        registry.addHandler(handler, "/heart/current")
                .addInterceptors(new WebSocketInterceptor())
                .setAllowedOrigins("*");
    }
}
```

</br>
</br>

```java
public class WebSocketInterceptor implements HandshakeInterceptor {

    protected Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, org.springframework.web.socket.WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        // 将ServerHttpRequest转换成request请求相关的类，用来获取request域中的用户信息
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpServletRequest httpRequest = servletRequest.getServletRequest();
            // todo  用户验证
        }
        logger.info("beforeHandshake完成");
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, org.springframework.web.socket.WebSocketHandler wsHandler, Exception exception) {
        logger.info("afterHandshake完成");
    }
}
```

</br>
</br>

```java
/**
 * 即时心跳的 webSocket消息处理
 *
 * @author by SunYuXing on 2019-01-16.
 */
@Component
public class HeartSocketHandler extends TextWebSocketHandler {

    private static Logger logger = LoggerFactory.getLogger(HeartSocketHandler.class);

    private List<WebSocketSession> sessions = new CopyOnWriteArrayList<>();

    @Reference(version = CommonConstant.VERSION)
    private RedisService redis;

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) {
        String userId = message.getPayload();
        if (CheckUtil.isNotEmpty(userId)) {
            String heart = redis.get(userId);
            sendMessageToUser(session, new TextMessage(heart == null ? "0" : heart));
        }
    }


    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        logger.info("Connected ... " + session.getId());
        sessions.add(session);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        if (session.isOpen()) {
            session.close();
        }
        sessions.remove(session);
        logger.warn("Session {} closed , code:{} because of {}", session.getId(),
                status.getCode(), status.getReason());
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable throwable) throws Exception {
        logger.error("error occur at sender " + session, throwable);
    }

    /**
     * 给所有的用户发送消息
     */
    private void sendMessagesToUsers(TextMessage message) {
        for (WebSocketSession user : sessions) {
            try {
                // isOpen()在线就发送
                if (user.isOpen()) {
                    user.sendMessage(message);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 发送消息给指定的用户
     */
    private void sendMessageToUser(WebSocketSession user, TextMessage message) {
        try {
            // 在线就发送
            if (user.isOpen()) {
                user.sendMessage(message);
            }
        } catch (IOException e) {
            logger.error("发送消息给指定的用户出错", e);
        }
    }
}
```