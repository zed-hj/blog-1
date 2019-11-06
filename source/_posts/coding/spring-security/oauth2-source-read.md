---
title: Spring Security OAuth2 源码阅读
date: 2019/11/5 21:14:00

categories: 
  - Spring Security
---

![Spring Security OAuth2 Password 方式获取 Token](http://cdn.talei.me/idea/spring-security-oauth2-password-mode.jpg)

<!--more-->

## Gateway

### UaaTokenEndpointClient

加密clientId和secret参数到请求头

```java
@Override
protected void addAuthentication(HttpHeaders reqHeaders, MultiValueMap<String, String> formParams) {
    reqHeaders.add("Authorization", getAuthorizationHeader());
}

/**
 * @return a Basic authorization header to be used to talk to UAA.
 */
protected String getAuthorizationHeader() {
    String clientId = getClientId();
    String clientSecret = getClientSecret();
    String authorization = clientId + ":" + clientSecret;
    return "Basic " + Base64Utils.encodeToString(authorization.getBytes(StandardCharsets.UTF_8));
}
```

### OAuth2TokenEndpointClientAdapter

微服务调用uaa,password方式认证获取token

```java
/**
 * Sends a password grant to the token endpoint.
 *
 * @param params the username to authenticate. his password.
 * @return the access token.
 */
@Override
public OAuth2AccessToken sendPasswordGrant(Map<String, String> params) {
    HttpHeaders reqHeaders = new HttpHeaders();
    reqHeaders.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    MultiValueMap<String, String> formParams = new LinkedMultiValueMap<>();
    String username = params.get("username");
    String password = params.get("password");
    formParams.set("username", username);
    formParams.set("password", password);
    formParams.set("grant_type", "password");
    addAuthentication(reqHeaders, formParams);
    HttpEntity<MultiValueMap<String, String>> entity = new HttpEntity<>(formParams, reqHeaders);
    ResponseEntity<OAuth2AccessToken>
        responseEntity = restTemplate.postForEntity(getTokenEndpoint(), entity, OAuth2AccessToken.class);
    if (responseEntity.getStatusCode() != HttpStatus.OK) {
        log.debug("failed to authenticate user with OAuth2 token endpoint, status: {}", responseEntity.getStatusCodeValue());
        throw new HttpClientErrorException(responseEntity.getStatusCode());
    }
    return responseEntity.getBody();
}
```

## Uaa
 
### TokenEndpoint

令牌请求的端点,客户端携带 grant_type 参数 (例如 'authorization_code') 和由授权类型确定的其他参数的请求。

支持的授予类型由 setTokenGranter 提供。

根据客户端携带的 grant_type 参数,交由匹配的 TokenGranter 去授予Token。

客户端必须使用 Authentication 进行身份验证才能访问此端点,并且客户端 id 是从身份验证令牌中提取的。

就是说必须在请求头里带上 Authorization,值为 clientId、secret base64加密,参数认证通过才能访问该端点。

```java
@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
    // 请求头携带的clientId,secret参数有误,拒绝访问.
    if (!(principal instanceof Authentication)) {
        throw new InsufficientAuthenticationException(
                "There is no client authentication. Try adding an appropriate authentication filter.");
    }
    String clientId = getClientId(principal);
    ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);
    TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedC
    if (clientId != null && !clientId.equals("")) {
        // Only validate the client details if a client authenticated during this
        // request.
        if (!clientId.equals(tokenRequest.getClientId())) {
            // double check to make sure that the client ID in the token request is the same as that in
            // authenticated client
            throw new InvalidClientException("Given client ID does not match authenticated client");
        }
    }
    if (authenticatedClient != null) {
        oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
    }
    if (!StringUtils.hasText(tokenRequest.getGrantType())) {
        throw new InvalidRequestException("Missing grant type");
    }
    if (tokenRequest.getGrantType().equals("implicit")) {
        throw new InvalidGrantException("Implicit grant type not supported from token endpoint");
    }
    if (isAuthCodeRequest(parameters)) {
        // The scope was requested or determined during the authorization step
        if (!tokenRequest.getScope().isEmpty()) {
            logger.debug("Clearing scope of incoming token request");
            tokenRequest.setScope(Collections.<String> emptySet());
        }
    }
    if (isRefreshTokenRequest(parameters)) {
        // A refresh token has its own default scopes, so we should ignore any added by the factory her
        tokenRequest.setScope(OAuth2Utils.parseParameterList(parameters.get(OAuth2Utils.SCOPE)));
    }

    // 根据 grant_type 委派匹配的 token 授予者去授予 token
    OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
    if (token == null) {
        throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType(
    }
    return getResponse(token);
}
```

### CompositeTokenGranter

该类维护一个 List<TokenGranter>, 遍历根据 `grant_type` 匹配 `TokenGranter` 去授予Token。

```java
/**
 * @author Dave Syer
 */
public class CompositeTokenGranter implements TokenGranter {

    private final List<TokenGranter> tokenGranters;

    public CompositeTokenGranter(List<TokenGranter> tokenGranters) {
        this.tokenGranters = new ArrayList<TokenGranter>(tokenGranters);
    }
    
    public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
        for (TokenGranter granter : tokenGranters) {
            OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
            if (grant!=null) {
                return grant;
            }
        }
        return null;
    }
    
    public void addTokenGranter(TokenGranter tokenGranter) {
        if (tokenGranter == null) {
            throw new IllegalArgumentException("Token granter is null");
        }
        tokenGranters.add(tokenGranter);
    }
}
```

### ResourceOwnerPasswordTokenGranter

用户名密码方式授予Token `grant_type: password`

```java
@Override
protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {
    Map<String, String> parameters = new LinkedHashMap<String, String>(tokenRequest.getRequestParameters());
    String username = parameters.get("username");
    String password = parameters.get("password");
    // Protect from downstream leaks of password
    parameters.remove("password");
    Authentication userAuth = new UsernamePasswordAuthenticationToken(username, password);
    ((AbstractAuthenticationToken) userAuth).setDetails(parameters);
    try {
        userAuth = authenticationManager.authenticate(userAuth);
    }
    catch (AccountStatusException ase) {
        //covers expired, locked, disabled cases (mentioned in section 5.2, draft 31)
        throw new InvalidGrantException(ase.getMessage());
    }
    catch (BadCredentialsException e) {
        // If the username/password are wrong the spec says we should send 400/invalid grant
        throw new InvalidGrantException(e.getMessage());
    }
    if (userAuth == null || !userAuth.isAuthenticated()) {
        throw new InvalidGrantException("Could not authenticate user: " + username);
    }
    
    OAuth2Request storedOAuth2Request = getRequestFactory().createOAuth2Request(client, tokenRequest);        
    return new OAuth2Authentication(storedOAuth2Request, userAuth);
}
```

### AuthenticationManagerDelegator

延迟构建认证管理器,确保配置完整,然后交由委托器去委托具体的认证管理器去认证。

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    if (delegate != null) {
        return delegate.authenticate(authentication);
    }
    synchronized (delegateMonitor) {
        if (delegate == null) {
            delegate = this.delegateBuilder.getObject();
            this.delegateBuilder = null;
        }
    }
    return delegate.authenticate(authentication);
}
```

### ProviderManager

委托认证管理器,遍历 providers,根据 Authentication 实现类去匹配对应的认证提供者(AuthenticationProvider)提供认证。

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    AuthenticationException lastException = null;
    AuthenticationException parentException = null;
    Authentication result = null;
    Authentication parentResult = null;
    boolean debug = logger.isDebugEnabled();

    // 遍历匹配对应的认证提供者
    for (AuthenticationProvider provider : getProviders()) {
        if (!provider.supports(toTest)) {
            continue;
        }

        if (debug) {
            logger.debug("Authentication attempt using "
                    + provider.getClass().getName());
        }

        try {
            result = provider.authenticate(authentication);

            if (result != null) {
                copyDetails(authentication, result);
                break;
            }
        }
        catch (AccountStatusException e) {
            prepareException(e, authentication);
            // SEC-546: Avoid polling additional providers if auth failure is due to
            // invalid account status
            throw e;
        }
        catch (InternalAuthenticationServiceException e) {
            prepareException(e, authentication);
            throw e;
        }
        catch (AuthenticationException e) {
            lastException = e;
        }
    }

    if (result == null && parent != null) {
        // Allow the parent to try.
        try {
            // 自己认证不了,让父级去认证
            result = parentResult = parent.authenticate(authentication);
        }
        catch (ProviderNotFoundException e) {
            // ignore as we will throw below if no other exception occurred prior to
            // calling parent and the parent
            // may throw ProviderNotFound even though a provider in the child already
            // handled the request
        }
        catch (AuthenticationException e) {
            lastException = parentException = e;
        }
    }

    if (result != null) {
        if (eraseCredentialsAfterAuthentication
                && (result instanceof CredentialsContainer)) {
            // Authentication is complete. Remove credentials and other secret data
            // from authentication
            ((CredentialsContainer) result).eraseCredentials();
        }

        // If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
        // This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
        if (parentResult == null) {
            eventPublisher.publishAuthenticationSuccess(result);
        }
        return result;
    }

    // Parent was null, or didn't authenticate (or throw an exception).

    if (lastException == null) {
        lastException = new ProviderNotFoundException(messages.getMessage(
                "ProviderManager.providerNotFound",
                new Object[] { toTest.getName() },
                "No AuthenticationProvider found for {0}"));
    }

    // If the parent AuthenticationManager was attempted and failed than it will publish an AbstractAuthenticationFailureEvent
    // This check prevents a duplicate AbstractAuthenticationFailureEvent if the parent AuthenticationManager already published it
    if (parentException == null) {
        prepareException(lastException, authentication);
    }

    throw lastException;
}
```

### AbstractUserDetailsAuthenticationProvider

用户详细认证抽象类

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
            messages.getMessage(
                    "AbstractUserDetailsAuthenticationProvider.onlySupports",
                    "Only UsernamePasswordAuthenticationToken is supported"));

    // Determine username
    String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
            : authentication.getName();

    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);

    if (user == null) {
        cacheWasUsed = false;

        try {
            // 拉取用户信息
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (UsernameNotFoundException notFound) {
            logger.debug("User '" + username + "' not found");

            if (hideUserNotFoundExceptions) {
                throw new BadCredentialsException(messages.getMessage(
                        "AbstractUserDetailsAuthenticationProvider.badCredentials",
                        "Bad credentials"));
            }
            else {
                throw notFound;
            }
        }

        Assert.notNull(user,
                "retrieveUser returned null - a violation of the interface contract");
    }

    try {
        preAuthenticationChecks.check(user);
    
        // 验证密码
        additionalAuthenticationChecks(user,
                (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (AuthenticationException exception) {
        if (cacheWasUsed) {
            // There was a problem, so try again after checking
            // we're using latest data (i.e. not from the cache)
            cacheWasUsed = false;
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
            preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        else {
            throw exception;
        }
    }

    postAuthenticationChecks.check(user);

    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }

    Object principalToReturn = user;

    if (forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }

    // 重新组装认证对象
    return createSuccessAuthentication(principalToReturn, authentication, user);
}
```

### DaoAuthenticationProvider

匹配 UsernamePasswordAuthenticationToken 方式的认证提供者。

```java
// 根据用户名加载用户详细数据
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    prepareTimingAttackProtection();
    try {
        // 根据用户名加载用户详细数据
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException("UserDetailsService returned null, which is an interface contract violation");
        }
        return loadedUser;
    }
    catch (UsernameNotFoundException ex) {
        mitigateAgainstTimingAttack(authentication);
        throw ex;
    }
    catch (InternalAuthenticationServiceException ex) {
        throw ex;
    }
    catch (Exception ex) {
        throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    }
}

// 验证密码
protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    if (authentication.getCredentials() == null) {
        logger.debug("Authentication failed: no credentials provided");
        throw new BadCredentialsException(messages.getMessage(
                "AbstractUserDetailsAuthenticationProvider.badCredentials",
                "Bad credentials"));
    }
    
    String presentedPassword = authentication.getCredentials().toString();
    // 验证密码
    if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
        logger.debug("Authentication failed: password does not match stored value");
        throw new BadCredentialsException(messages.getMessage(
                "AbstractUserDetailsAuthenticationProvider.badCredentials",
                "Bad credentials"));
    }
}

// 认证成功后重新组装认证对象
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication, UserDetails user) {
    // Ensure we return the original credentials the user supplied,
    // so subsequent attempts are successful even with encoded passwords.
    // Also ensure we return the original getDetails(), so that future
    // authentication events after cache expiry contain the details
    UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
            principal, authentication.getCredentials(),
            authoritiesMapper.mapAuthorities(user.getAuthorities()));
    result.setDetails(authentication.getDetails());
    return result;
}
```

## 结尾

阅读源码发现 gateway 是调用 uaa TokenEndpoint 端点的 `/oauth/token` 接口去认证获取Token的。

CompositeTokenGranter(包含了其他授予者的授予者)是根据 gateway 传的 grant_type 参数决定调用哪个授予者去授予Token的。

```java
CompositeTokenGranter.java

// 维护了其他授予者
private final List<TokenGranter> tokenGranters;

public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
    // 自己不做事,安排合适的人去做
    for (TokenGranter granter : tokenGranters) {
        OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
        if (grant!=null) {
            return grant;
        }
    }
    return null;
}

AbstractTokenGranter.java

// 客户端指定用什么方式授予Token
public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
    // 每个实现类都要通过父类构造传入 grantType参数,表示自己能够处理。
    if (!this.grantType.equals(grantType)) {
        return null;
    }
    ...
}
```

ProviderManager 是根据 Authentication 具体实现类去决定调用哪个 AuthenticationProvider 实现类去认证的。

```java
ProviderManager.java

// 遍历匹配对应的认证提供者
for (AuthenticationProvider provider : getProviders()) {
    if (!provider.supports(toTest)) {
        continue;
    }
}

AbstractUserDetailsAuthenticationProvider.java

// 支持入参为 UsernamePasswordAuthenticationToken 类型的认证 
public boolean supports(Class<?> authentication) {
    return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
}
```


| grant_type | TokenGranter | Authentication | AuthenticationProvider |
| --------- | ------------ | -------------- | ---------------------- |
| password | ResourceOwnerPasswordTokenGranter | UsernamePasswordAuthenticationToken | DaoAuthenticationProvider |


可以发现几个接口都是配套使用的，我们去实现这三个接口即可扩展出自定义授权方式。