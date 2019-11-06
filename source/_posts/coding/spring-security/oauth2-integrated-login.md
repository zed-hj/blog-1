---
title: Spring Security OAuth2 集成登录
date: 2019/11/6 21:14:00

categories: 
  - Spring Security
---

![微信扫码登陆](http://cdn.talei.me/idea/wechat-qrcode-login.jpeg)

<!--more-->

1. 扩展集成登录授予者(integrated-login)
2. 授予者根据 type 执行具体的认证策略

基于策略模式可以轻松新增任意登录方式。

## 扩展集成登录TokenGranter

### LoginTypeEnums

登录类型

```java
public enum LoginTypeEnums {
    SMS("sms", "验证码登录"),
    WECHAT("wechat", "微信扫码登陆");

    private String key;
    private String val;

    LoginTypeEnums(String key, String val) {
        this.key = key;
        this.val = val;
    }

    public String getKey() {
        return key;
    }

    public String getVal() {
        return val;
    }
}
```

### WechatOAuth2Helper

微信授权帮助类

```java

@Component
public class WechatOAuth2Helper {

    private static final String GET_TOKEN = "https://api.weixin.qq.com/sns/oauth2/access_token?appid=%s&secret=%s&code=%s&grant_type=authorization_code";
    private static final String REFRESH_TOKEN = "https://api.weixin.qq.com/sns/oauth2/refresh_token?appid=%s&grant_type=refresh_token&refresh_token=%s";
    private static final String VALIDATE_TOKEN = "https://api.weixin.qq.com/sns/auth?access_token=%S&openid=%s";
    private static final String GET_USER_INFO = "https://api.weixin.qq.com/sns/userinfo?access_token=%s&openid=%s";

    @Resource
    private ApplicationProperties applicationProperties;

    public WechatTokenResult getToken(String code) {
        String body = HttpRequest.get(String.format(
                GET_TOKEN,
                applicationProperties.getWechat().getAppId(),
                applicationProperties.getWechat().getSecret(),
                code))
                .execute()
                .body();

        return JSON.parseObject(body, WechatTokenResult.class);
    }

    public WechatTokenResult refreshToken(String refreshToken) {
        String body = HttpRequest.get(String.format(
                REFRESH_TOKEN,
                applicationProperties.getWechat().getAppId(),
                refreshToken))
                .execute()
                .body();

        return JSON.parseObject(body, WechatTokenResult.class);
    }

    public WechatResult validateToken(String accessToken, String openId) {
        String body = HttpRequest.get(String.format(
                VALIDATE_TOKEN,
                accessToken,
                openId))
                .execute()
                .body();

        return JSON.parseObject(body, WechatResult.class);
    }

    public WechatUserInfo getUserInfo(String accessToken, String openId) {
        String body = HttpRequest.get(String.format(
                GET_USER_INFO,
                accessToken,
                openId))
                .execute()
                .body();

        return JSON.parseObject(body, WechatUserInfo.class);
    }

    @Data
    private static class WechatResult implements Serializable {
        private static final long serialVersionUID = -3957124734160435542L;

        // failure
        private int errcode;
        private String errmsg;
    }

    @Data
    @EqualsAndHashCode(callSuper = true)
    public static class WechatTokenResult extends WechatResult {
        private static final long serialVersionUID = -962164874452135672L;

        private String access_token;
        private long expires_in;
        private String refresh_token;
        private String openid;
        private String scope;
    }

    @Data
    @EqualsAndHashCode(callSuper = true)
    public static class WechatUserInfo extends WechatResult {
        private static final long serialVersionUID = 8852967139746957255L;

        private String openid;
        private String nickname;
        private int sex;
        private String province;
        private String city;
        private String country;
        private String headimgurl;
        private String unionid;
        private List<String> privilege;
    }
}
```

### LoginValidatedHelper

根据 type 执行对应的认证逻辑。

```java
@Component
public class LoginValidatedHelper {

    private Map<String, Function<Map<String, String>, Authentication>> strategyMap = new HashMap<>();

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Resource
    private UserConnectionRepository userConnectionRepository;

    @Resource
    private UserService userService;

    @PostConstruct
    public void init() {
        strategyMap.put(LoginTypeEnums.SMS.getKey(), smsValidated());
        strategyMap.put(LoginTypeEnums.WECHAT.getKey(), wechatValidated());
    }

    /**
     * 获取策略
     */
    public Authentication exec(String type, Map<String, String> params) {
        Function<Map<String, String>, Authentication> strategy = strategyMap.get(type);
        if (strategy == null) {
            throw new BadRequestAlertException("invalid login type", "type", "invalidtype");
        }
        return strategy.apply(params);
    }

    /**
     * 短信验证码登录验证
     *
     * @return
     */
    private Function<Map<String, String>, Authentication> smsValidated() {
        return (params) -> {
            String username = params.get("username");
            String smsCode = params.get("smsCode");

            String realCode = stringRedisTemplate.opsForValue().get(LOGIN_USER_SMS + username);
            if (StringUtils.isBlank(realCode) || !realCode.equals(smsCode)) {
                throw new BadCredentialsException("sms code error");
            }

            LoginAuthenticationToken userAuth = new LoginAuthenticationToken(username);
            userAuth.setDetails(params);
            return userAuth;
        };
    }

    /**
     * 微信扫码登陆
     *
     * @return
     */
    private Function<Map<String, String>, Authentication> wechatValidated() {
        return (params) -> {
            String openId = params.get("openId");

            UserConnection userConn = userConnectionRepository.findByProviderAndOpenId(SocialProviderEnum.WECHAT.getKey(), openId);
            if (userConn == null || StringUtils.isBlank(userConn.getLogin()) || userConn.getLogin().equals("none")) {
                throw new BadCredentialsException("wechat open id not exists");
            }

            User user = userService.findByLogin(userConn.getLogin()).orElseThrow(() -> new DataNotFoundException("user not found"));
            LoginAuthenticationToken userAuth = new LoginAuthenticationToken(user.getLogin());
            userAuth.setDetails(params);
            return userAuth;
        };
    }
}
```

### LoginTokenGranter 

集成登录Token授予者,处理所有集成登录

```java
public class LoginTokenGranter extends AbstractTokenGranter {

    private static final String GRANT_TYPE = "integrated-login";

    private LoginValidatedHelper loginValidatedHelper;

    // 其他子类这里注入的是 AuthenticationManager, 我这里很明确会使用集成登录认证
    private LoginAuthenticationProvider loginAuthenticationProvider;

    public LoginTokenGranter(AuthorizationServerTokenServices tokenServices,
                             ClientDetailsService clientDetailsService,
                             OAuth2RequestFactory requestFactory,
                             UserDetailsService userDetailsService,
                             LoginValidatedHelper loginValidatedHelper) {
        this(tokenServices, clientDetailsService, requestFactory, GRANT_TYPE, userDetailsService, loginValidatedHelper);
    }

    protected LoginTokenGranter(AuthorizationServerTokenServices tokenServices,
                                ClientDetailsService clientDetailsService,
                                OAuth2RequestFactory requestFactory, String grantType,
                                UserDetailsService userDetailsService,
                                LoginValidatedHelper loginValidatedHelper) {
        super(tokenServices, clientDetailsService, requestFactory, grantType);

        loginAuthenticationProvider = new LoginAuthenticationProvider();
        loginAuthenticationProvider.setUserDetailsService(userDetailsService);

        this.loginValidatedHelper = loginValidatedHelper;
    }

    @Override
    public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
        return super.grant(grantType, tokenRequest);
    }

    @Override
    protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {

        Map<String, String> parameters = new LinkedHashMap<>(tokenRequest.getRequestParameters());

        Authentication userAuth;
        try {
            /*
             * 没报异常则表示验证成功
             */
            userAuth = loginValidatedHelper.exec(parameters.get("type"), parameters);
            userAuth = loginAuthenticationProvider.authenticate(userAuth);
        } catch (AccountStatusException ase) {
            //covers expired, locked, disabled cases (mentioned in section 5.2, draft 31)
            throw new InvalidGrantException(ase.getMessage());
        } catch (BadCredentialsException e) {
            // If the username/password are wrong the spec says we should send 400/invalid grant
            throw new InvalidGrantException(e.getMessage());
        }
        if (userAuth == null || !userAuth.isAuthenticated()) {
            throw new InvalidGrantException("Could not authenticate user");
        }

        OAuth2Request storedOAuth2Request = getRequestFactory().createOAuth2Request(client, tokenRequest);
        return new OAuth2Authentication(storedOAuth2Request, userAuth);
    }
}
```

### LoginAuthenticationToken

```java
public class LoginAuthenticationToken extends AbstractAuthenticationToken {
    private static final long serialVersionUID = -8358197885914919681L;

    private final Object principal;

    public LoginAuthenticationToken(Object principal) {
        super(null);
        this.principal = principal;
        setAuthenticated(false);
    }

    public LoginAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        super.setAuthenticated(true);
    }


    @Override
    public Object getCredentials() {
        return null;
    }

    @Override
    public Object getPrincipal() {
        return this.principal;
    }

    @Override
    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException("Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }

        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}
```

### LoginAuthenticationProvider

```java
@Slf4j
@Component
public class LoginAuthenticationProvider implements AuthenticationProvider {

    protected boolean hideUserNotFoundExceptions = true;

    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        LoginAuthenticationToken authenticationToken = (LoginAuthenticationToken) authentication;
        
        // 根据用户名获取用户直接认证成功
        UserDetails user = userDetailsService.loadUserByUsername((String) authenticationToken.getPrincipal());

        if (user == null) {
            if (hideUserNotFoundExceptions) {
                log.error("user name {} is null", authenticationToken.getPrincipal());
            } else {
                throw new BadCredentialsException("user name is null");
            }
        }
        LoginAuthenticationToken authenticationResult = new LoginAuthenticationToken(user, user.getAuthorities());
        authenticationResult.setDetails(authenticationToken.getDetails());
        return authenticationResult;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return LoginAuthenticationToken.class.isAssignableFrom(authentication);
    }

    public UserDetailsService getUserDetailsService() {
        return userDetailsService;
    }

    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }
}
```

### UaaConfiguration

注册自定义集成登录

```java
@Configuration
@EnableAuthorizationServer
public class UaaConfiguration extends AuthorizationServerConfigurerAdapter implements ApplicationContextAware {
    
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .authorizedGrantTypes("implicit", "refresh_token", "password", "authorization_code", "integrated-login"); // 将 集成登录方式加进去
        ...    
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .tokenGranter(tokenGranter(endpoints));              // 默认的 + 自定义的授予者
        ...
    }

    @Resource
    private UserDetailsService userDetailsService;

    @Resource
    private LoginValidatedHelper loginValidatedHelper;

    private TokenGranter tokenGranter(final AuthorizationServerEndpointsConfigurer endpoints) {
        List<TokenGranter> granters = new ArrayList<>(Arrays.asList(endpoints.getTokenGranter()));// 获取默认的granter集合
        // 将自定义的加进list
        granters.add(new LoginTokenGranter(endpoints.getTokenServices(), endpoints.getClientDetailsService(), endpoints.getOAuth2RequestFactory(), userDetailsService, loginValidatedHelper));
        return new CompositeTokenGranter(granters);
    }

    ...
}

```

## UserConnection

存储第三方平台用户与系统用户的关联数据。

### DDL

```sql
create table uaa_user_connection (
    login                   varchar(50) default 'none' not null,
    provider                varchar(50)                not null,
    provider_open_id        varchar(50)                not null,
    provider_global_user_id varchar(50)                null,
    access_token            varchar(512)               not null,
    refresh_token           varchar(512)               null,
    expire_time             bigint                     null,
    detailed                varchar(1024)              null,
    primary key (login, provider, provider_open_id)
);
```

### SocialProviderEnum

社交平台枚举

```java
public enum SocialProviderEnum {

    WECHAT("wechat", "微信");

    @EnumValue
    @JsonValue
    private String key;
    private String info;

    SocialProviderEnum(String key, String info) {
        this.key = key;
        this.info = info;
    }

    public static SocialProviderEnum ofKey(String key) {
        for (SocialProviderEnum value : SocialProviderEnum.values()) {
            if (value.getKey().equals(key)) {
                return value;
            }
        }
        throw new RuntimeException("enum key not match");
    }

    public String getKey() {
        return key;
    }

    public String getInfo() {
        return info;
    }
}
```

### UserConnection

Entity class

```java
public class UserConnection implements Serializable {
    private static final long serialVersionUID = -2542143868410372668L;

    private String login;
    private SocialProviderEnum provider;
    private String providerOpenId;
    private String providerGlobalUserId;
    private String accessToken;
    private String refreshToken;
    private Long expireTime;
    private String detailed;
}
```

### UserConnectionRepository

```java
@Mapper
public interface UserConnectionRepository extends BaseMapper<UserConnection> {

    @Select("SELECT * FROM uaa_user_connection WHERE provider = #{provider} AND provider_open_id = #{openId}")
    UserConnection findByProviderAndOpenId(@Param("provider") String provider, @Param("openId") String openId);
}
```

### UserConnectionService

```java
public interface UserConnectionService {

    /*
     * 根据code获取用户信息,并返回关联标识
     */
    WechatGetOpenIdResultDTO getWechatOpenId(String code);

    /**
     * 关联系统账户
     */
    void associationAccount(AssociationAccountParam param);
}
```

### UserConnectionServiceImpl

```java
@Slf4j
@Component
public class UserConnectionServiceImpl implements UserConnectionService {

    @Resource
    private WechatOAuth2Helper wechatOAuth2Helper;

    @Resource
    private UserConnectionRepository userConnectionRepository;

    @Resource
    private UserService userService;

    @Override
    public WechatGetOpenIdResultDTO getWechatOpenId(String code) {
        /*
         * TODO
         * 1. 不存在接入记录
         *  - 获取用户信息
         *  - 保存接入记录
         *  - 重定向到关联已有账户或创建新用户页面
         *
         * 2. 存在记录
         *  - 返回关联标识和 open_id
         */
        WechatOAuth2Helper.WechatTokenResult token = wechatOAuth2Helper.getToken(code);
        UserConnection userConnectionExists = userConnectionRepository.findByProviderAndOpenId(SocialProviderEnum.WECHAT.getKey(), token.getOpenid());
        if (userConnectionExists == null) {
            WechatOAuth2Helper.WechatUserInfo userInfo = wechatOAuth2Helper.getUserInfo(token.getAccess_token(), token.getOpenid());

            UserConnection userConnection = new UserConnection()
                    .setProvider(SocialProviderEnum.WECHAT)
                    .setProviderOpenId(userInfo.getOpenid())
                    .setProviderGlobalUserId(userInfo.getUnionid())
                    .setAccessToken(token.getAccess_token())
                    .setRefreshToken(token.getRefresh_token())
                    .setExpireTime(token.getExpires_in())
                    .setDetailed(JSON.toJSONString(userInfo));

            userConnectionRepository.insert(userConnection);

            return new WechatGetOpenIdResultDTO()
                    .setProvider(SocialProviderEnum.WECHAT.getKey())
                    .setOpenId(userInfo.getOpenid());
        }

        return new WechatGetOpenIdResultDTO()
                .setProvider(SocialProviderEnum.WECHAT.getKey())
                .setOpenId(userConnectionExists.getProviderOpenId())
                .setAssociated(StrUtil.isNotBlank(userConnectionExists.getLogin()) && !userConnectionExists.getLogin().equals("none"));
    }

    @Override
    public void associationAccount(AssociationAccountParam param) {
        UserConnection userConnection = userConnectionRepository.findByProviderAndOpenId(SocialProviderEnum.WECHAT.getKey(), param.getOpenId());
        if (userConnection == null) {
            throw new DataNotFoundException("User connection not found");
        }
        if (StrUtil.isNotBlank(userConnection.getLogin()) && !userConnection.getLogin().equals("none")) {
            throw new IllegalOperationException("The account already association other user");
        } else {
            userService.findByLogin(param.getLogin()).orElseThrow(() -> new DataNotFoundException("User login not found"));
            userConnection.setLogin(param.getLogin());
            userConnectionRepository.update(userConnection, new QueryWrapper<>(new UserConnection()
                    .setProvider(SocialProviderEnum.ofKey(param.getProvider()))
                    .setProviderOpenId(param.getOpenId())));
        }
    }
}
```

### UserConnectionResource

```java
@Validated
@RestController
@RequestMapping("/api")
public class UserConnectionResource {

    @Resource
    private UserConnectionService userConnectionService;

    @PostMapping("/association-account")
    public ResponseEntity<Boolean> associationAccount(@RequestBody @Validated AssociationAccountParam param) {
        userConnectionService.associationAccount(param);
        return ResponseEntity.ok(true);
    }

    @GetMapping("/service/social/wechat/{code}")
    public ResponseEntity<WechatGetOpenIdResultDTO> getWechatOpenId(@PathVariable("code") String code) {
        return ResponseEntity.ok(userConnectionService.getWechatOpenId(code));
    }
}
```

### AssociationAccountParam

社交平台账户关联系统账户的参数类

```java
@Data
@Accessors(chain = true)
public class AssociationAccountParam extends BaseRequest {
    private static final long serialVersionUID = 8089720671889219334L;

    @NotBlank
    private String provider;
    @NotBlank
    private String openId;
}
```

## 结尾

1. 在 UaaConfiguration 类注册新增的集成登录(LoginTokenGranter)和注册 `grant_type:` `integrated-login`
2. 所有登录方式(sms,wechat...)复用同一个 TokenGranter,TokenGranter 在认证的时候再根据登录方式执行具体的认证逻辑。