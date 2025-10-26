# 博客-redis

## 使用场景
和之前的项目一样，在博客项目中也使用redis作为缓存。

1. 缓存管理

    关于页面缓存：使用键ABOUT缓存关于页面内容

    用户区域统计：使用键USER_AREA缓存用户区域统计数据

    访客区域统计：使用键VISITOR_AREA缓存访客区域统计数据

2. 访问统计与限制

    博客访问计数：使用键BLOG_VIEWS_COUNT记录博客总访问量

    文章访问计数：使用键ARTICLE_VIEWS_COUNT记录各文章访问量（使用ZSet数据结构）

    唯一访客标识：使用键UNIQUE_VISITOR存储唯一访客信息

3. 用户认证与授权

    登录用户信息：使用键LOGIN_USER存储登录用户信息

    验证码存储：使用键USER_CODE_KEY存储邮件验证码，15分钟过期

4. 接口限流

    在AccessLimitInterceptor.java中使用Redis实现接口访问频率限制

5. 文章访问权限控制

    使用键ARTICLE_ACCESS存储用户对加密文章的访问权限

## 功能实现
 
1. 用户认证

   对于用户的登录信息，采用redis的hash的方式保存，field为用户ID，value为UserDetailsDTO对象。在解析用户信息时，首先从请求头获取JWT Token并解析出用户ID，1r h根据用户ID从Redis Hash中获取用户详细信息，接着会有一个续期操作检查Token是否需要续期（如果剩余时间小于20分钟）。

   ```java

    @Override
    public void refreshToken(UserDetailsDTO userDetailsDTO) {
        LocalDateTime currentTime = LocalDateTime.now();
        userDetailsDTO.setExpireTime(currentTime.plusSeconds(EXPIRE_TIME));
        String userId = userDetailsDTO.getId().toString();
        redisService.hSet(LOGIN_USER, userId, userDetailsDTO, EXPIRE_TIME);
    }

    @Override
    public void renewToken(UserDetailsDTO userDetailsDTO) {
        LocalDateTime expireTime = userDetailsDTO.getExpireTime();
        LocalDateTime currentTime = LocalDateTime.now();
        if (Duration.between(currentTime, expireTime).toMinutes() <= TWENTY_MINUTES) {
            refreshToken(userDetailsDTO);
        }
    }
    ```

   对于登录的验证码，使用code:邮箱的方式存在redis中，同时将消息投递到消息队列。

   ```java
   @Override
    public void sendCode(String username) {
        if (!checkEmail(username)) {
            throw new BizException("请输入正确邮箱");
        }
        String code = getRandomCode();
        Map<String, Object> map = new HashMap<>();
        map.put("content", "您的验证码为 " + code + " 有效期15分钟，请不要告诉他人哦！");
        EmailDTO emailDTO = EmailDTO.builder()
                .email(username)
                .subject(CommonConstant.CAPTCHA)
                .template("common.html")
                .commentMap(map)
                .build();
        rabbitTemplate.convertAndSend(EMAIL_EXCHANGE, "*", new Message(JSON.toJSONBytes(emailDTO), new MessageProperties()));
        redisService.set(USER_CODE_KEY + username, code, CODE_EXPIRE_TIME);
    }
    ```



2. 接口限流

   接口限流是通过注解和redis共同实现的，通过注解在方法上设置对应的时间和访问次数限制，每次访问通过redis的原子自增的方式`incrExpire1`来增加访问次数，再判断。

   ```java
   @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            AccessLimit accessLimit = handlerMethod.getMethodAnnotation(AccessLimit.class);
            if (accessLimit != null) {
                long seconds = accessLimit.seconds();
                int maxCount = accessLimit.maxCount();
                String key = IpUtil.getIpAddress(httpServletRequest) + "-" + handlerMethod.getMethod().getName();
                try {
                    long q = redisService.incrExpire(key, seconds);
                    if (q > maxCount) {
                        render(httpServletResponse, ResultVO.fail("请求过于频繁，" + seconds + "秒后再试"));
                        log.warn(key + "请求次数超过每" + seconds + "秒" + maxCount + "次");
                        return false;
                    }
                    return true;
                } catch (RedisConnectionFailureException e) {
                    log.warn("redis错误: " + e.getMessage());
                    return false;
                }
            }
        }
        return true;
    }
    ```



3. 部分字段的缓存

    使用键ABOUT缓存关于页面内容
    ```java
    public AboutDTO getAbout() {
        AboutDTO aboutDTO;
        Object about = redisService.get(ABOUT);
        if (Objects.nonNull(about)) {
            aboutDTO = JSON.parseObject(about.toString(), AboutDTO.class);
        } else {
            String content = aboutMapper.selectById(DEFAULT_ABOUT_ID).getContent();
            aboutDTO = JSON.parseObject(content, AboutDTO.class);
            redisService.set(ABOUT, content);
        }
        return aboutDTO;
    }

    @Override
    public AboutDTO getAbout() {
        AboutDTO aboutDTO;
        Object about = redisService.get(ABOUT);
        if (Objects.nonNull(about)) {
            aboutDTO = JSON.parseObject(about.toString(), AboutDTO.class);
        } else {
            String content = aboutMapper.selectById(DEFAULT_ABOUT_ID).getContent();
            aboutDTO = JSON.parseObject(content, AboutDTO.class);
            redisService.set(ABOUT, content);
        }
        return aboutDTO;
    }
```
获取时先去读取redis，写入的时候先删除缓存，在重新写入缓存
