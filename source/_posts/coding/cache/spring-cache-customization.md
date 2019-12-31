---
title: Spring Cache 自定义缓存时间

date: 2019/07/20 23:30:32

categories:
    - 缓存
---

@Cacheable 接口无法配置过期时间, 全局只能使用同一个过期时间很不灵活。

为此我们需要一个可灵活配置过期时间版的 @Cacheable 注解。

<!--more-->

### @ICacheable

通过自带注解的 cacheResolver 属性绑定一个 CacheResolver(缓存解决方) 子类。

不单单只是 @Cacheable 注解有 cacheResolver 属性, @CacheEvict、@CachePut 都可以定制化。

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.core.annotation.AliasFor;

import java.lang.annotation.*;
import java.util.concurrent.TimeUnit;

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Cacheable(cacheResolver = "iCacheResolver")
public @interface ICacheable {

    /**
     * 过期时间
     */
    int expires() default 500;

    TimeUnit unit() default TimeUnit.MINUTES;

    /**
     * @see Cacheable#value()
     */
    @AliasFor(annotation = Cacheable.class)
    String[] value() default {};

    /**
     * @see Cacheable#cacheNames()
     */
    @AliasFor(annotation = Cacheable.class)
    String[] cacheNames() default {};

    /**
     * @see Cacheable#key()
     */
    @AliasFor(annotation = Cacheable.class)
    String key() default "";

    /**
     * @see Cacheable#keyGenerator()
     */
    @AliasFor(annotation = Cacheable.class)
    String keyGenerator() default "";

    /**
     * @see Cacheable#cacheManager()
     */
    @AliasFor(annotation = Cacheable.class)
    String cacheManager() default "";

    /**
     * @see Cacheable#condition()
     */
    @AliasFor(annotation = Cacheable.class)
    String condition() default "";

    /**
     * @see Cacheable#unless()
     */
    @AliasFor(annotation = Cacheable.class)
    String unless() default "";

    /**
     * @see Cacheable#sync()
     */
    @AliasFor(annotation = Cacheable.class)
    boolean sync() default false;
}
```


### RedisCache

RedisCache 本身构造方法是 protected 级别的, 所以必须要继承它并对外暴露构造。

```java
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheWriter;

public class RedisCache extends org.springframework.data.redis.cache.RedisCache {

    public RedisCache(String name, RedisCacheWriter cacheWriter, RedisCacheConfiguration cacheConfig) {
        super(name, cacheWriter, cacheConfig);
    }
}
```

### ICacheResolver

在这儿实现具体的缓存逻辑。

```java
import afu.org.checkerframework.checker.nullness.qual.Nullable;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.interceptor.CacheOperationInvocationContext;
import org.springframework.cache.interceptor.SimpleCacheResolver;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheWriter;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.concurrent.TimeUnit;

public class ICacheResolver extends SimpleCacheResolver {

    private static final int DEFAULT_EXPIRED_SECONDS = 600;
    private static final Logger log = LoggerFactory.getLogger(ICacheResolver.class);

    private final RedisCacheWriter cacheWriter;

    private final RedisCacheConfiguration redisCacheConfiguration;


    public ICacheResolver(RedisCacheWriter cacheWriter, CacheManager cacheManager, RedisCacheConfiguration redisCacheConfiguration) {
        super(cacheManager);
        this.cacheWriter = cacheWriter;
        this.redisCacheConfiguration = redisCacheConfiguration;
    }

    @Override
    public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
        ICacheable cache = AnnotationUtils.findAnnotation(context.getMethod(), ICacheable.class);

        if (cache == null) {
            return super.resolveCaches(context);
        }
        return getCaches(context, cache);
    }

    private Collection<? extends Cache> getCaches(CacheOperationInvocationContext<?> context, ICacheable cacheAnnotations) {
        Collection<String> cacheNames = getCacheNames(context);
        if (cacheNames == null) {
            return Collections.emptyList();
        } else if (cacheNames.isEmpty()) {
            /*
             * 如果没有配置 cacheNames, 那么则默认使用 (包名.类目.方法名)
             */
            cacheNames.add(context.getTarget().getClass().getSimpleName() + ":" + context.getMethod().getName());
        }
        Collection<Cache> result = new ArrayList<>(cacheNames.size());
        int expires = cacheAnnotations.expires();

        Duration ttl = getDuration(cacheAnnotations, expires);
        for (String cacheName : cacheNames) {
            if (ttl == null) {
                throw new IllegalArgumentException("Cannot find cache named '" +
                        cacheName + "' for " + context.getOperation());
            }
            Cache cache = createRedisCache(cacheName, redisCacheConfiguration.entryTtl(ttl));
            result.add(cache);
        }

        return result;
    }

    private Duration getDuration(ICacheable cacheAnnotations, int expires) {
        Duration ttl;
        if (cacheAnnotations.expires() == 0) {
            ttl = Duration.ofSeconds(DEFAULT_EXPIRED_SECONDS);
        } else if (cacheAnnotations.unit().equals(TimeUnit.NANOSECONDS)) {
            ttl = Duration.ofNanos(expires);
        } else if (cacheAnnotations.unit().equals(TimeUnit.MILLISECONDS)) {
            ttl = Duration.ofMillis(expires);
        } else if (cacheAnnotations.unit().equals(TimeUnit.SECONDS)) {
            ttl = Duration.ofSeconds(expires);
        } else if (cacheAnnotations.unit().equals(TimeUnit.MINUTES)) {
            ttl = Duration.ofMinutes(expires);
        } else if (cacheAnnotations.unit().equals(TimeUnit.HOURS)) {
            ttl = Duration.ofHours(expires);
        } else if (cacheAnnotations.unit().equals(TimeUnit.DAYS)) {
            ttl = Duration.ofDays(expires);
        } else {
            /*
             * TODO
             *  不支持的时间单位
             *  使用默认过期时间
             */

            ttl = Duration.ofSeconds(DEFAULT_EXPIRED_SECONDS);
            log.warn("Unsupported time unit used: {}", cacheAnnotations.unit().toString());
        }
        return ttl;
    }

    private RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
        return new RedisCache(name, cacheWriter, cacheConfig);
    }
}
```

### CacheConfiguration

注入, 使 @ICacheable 能通过 cacheResolver 值找到对应的缓存解决方。

```java
@Configuration
@EnableCaching
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class CacheConfiguration {

    private final Logger log = LoggerFactory.getLogger(CacheConfiguration.class);

    // ignored other config...

    @Bean("iCacheResolver")
    @ConditionalOnMissingBean
    public ICacheResolver iCacheResolver(RedisCacheManager redisCacheManager) throws NoSuchFieldException, IllegalAccessException {
        RedisCacheConfiguration redisCacheConfiguration = redisCacheConfiguration();

        Field cacheWriterField = RedisCacheManager.class.getDeclaredField("cacheWriter");
        cacheWriterField.setAccessible(true);
        RedisCacheWriter cacheWriter = (RedisCacheWriter) cacheWriterField.get(redisCacheManager);

        return new ICacheResolver(cacheWriter, redisCacheManager, redisCacheConfiguration);
    }
}
```

配置好之后就可以用 @ICacheable 代替 @Cacheable 即可。
