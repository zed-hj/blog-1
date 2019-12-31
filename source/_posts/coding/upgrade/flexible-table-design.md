---
title: 表预留特性字段 & Entity 自动映射到 DTO

date: 2019/12/31 22:34:31

categories:
    - 弹性设计
---

## 思路

之前的项目中每个表都存在着一个 feature 字段, 但我只是把它当成一个候补字段去使用, 并没有将它发挥最大价值, 直到阅读时看到了了下面这段话: 

> 随着Kubernetes版本的持续升级, 一些资源对象会不断引入新的属性。为了在不影响当前功能的情况下引入对新特性的支持, 我们通常会采用下面两种典型的方法。
>
> 1. 在设计数据库表的时候, 在每个表中都添加一个很长的备注字段, 之后口占的数据以某种格式(如XML、JSON、简单字符串拼接)放入备注字段。因为数据库表的接口并没有发生变化, 所以此时程序的改动范围是最小的, 风险也更小, 但看起来不太美观。
>
> 2. 直接修改数据库表, 增加一个或多个新的列, 此时程序的改动范围较大, 风险更大, 但看起来比较美观。
>
> 显然, 两种方法都不完美。更加优雅的做法是, 先采用 **方法1** 实现这个新特性, 经过几个版本的迭代, 等新特性变得成熟稳定了以后, 可以在后续版本中采用 **方法2** 升级到正式版。

摘抄自: Kubernetes 权威指南(第四版)

思路有了, 那么我想让 String(entity) Convert Map(dto) 的代码只写一次。

<!--more-->

## Entity 自动映射到 DTO

自动完成 Entity、DTO 互转, 预留特性字段字段从 String 转成 Map, 免除每个模型都需要写重复的解析转换代码。

### 依赖

- [Hutool](https://hutool.cn/docs/#/)
- [Mapstruct](https://mapstruct.org/) [文档](https://mapstruct.org/documentation/stable/reference/html/) 编译时生成 EntityToDtoMapper 实现类, 自动映射属性值。

> qualifiedBy 指定某个属性和注解,表示属性的转换交由注解修饰的方法去实现

```xml
<properties>
    <hutool.version>5.0.5</hutool.version>
    <fastjson.version>1.2.46</fastjson.version>
    <mapstruct.version>1.3.1.Final</mapstruct.version>
</properties>

<dependencies>
    <!-- https://www.hutool.cn/docs/#/ -->
    <dependency>
        <groupId>cn.hutool</groupId>
        <artifactId>hutool-all</artifactId>
        <version>${hutool.version}</version>
    </dependency>
    
    <!-- https://mvnrepository.com/artifact/com.alibaba/fastjson -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>${fastjson.version}</version>
    </dependency>    

    <!-- https://mapstruct.org/ -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${mapstruct.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${mapstruct.version}</version>
            </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

### FeatureUtil

```java
import cn.hutool.core.map.MapUtil;
import cn.hutool.core.util.ReflectUtil;
import cn.hutool.core.util.StrUtil;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;

import java.util.Map;

@Slf4j
public class FeatureUtil {

    /**
     * 预留的弹性字段名, 会通过反射去获取。
     */
    private static final String FIELD_FEATURE = "feature";
    private static final String EMPTY_OBJECT = "{}";
    
    /**
     * 统一处理entity feature 转 dto feature
     */
    @FeatureToMap
    public static Map<String, Object> map(String feature) {
        if (StrUtil.isBlank(feature)) {
            return null;
        }

        try {
            return JSON.parseObject(feature).getInnerMap();
        } catch (Exception e) {
            log.error("[特性字段解析异常]: {}", e.getMessage());
            return null;
        }
    }
    
    /**
     * 统一处理dto feature 转 entity feature
     */
    @FeatureToJson
    public static String map(Map<String, Object> feature) {
        if (MapUtil.isEmpty(feature)) {
            return null;
        }

        return JSON.toJSONString(feature);
    }

    /**
     * 设置新值
     *
     * @param obj dto or domain
     * @param key field name
     * @param val field value
     * @param <T> dto or domain
     * @return dto or domain
     */
    public static <T> T putObject(T obj, String key, Object val) {
        String feature = (String) ReflectUtil.getFieldValue(obj, FIELD_FEATURE);
        if (StrUtil.isBlank(feature)) {
            feature = EMPTY_OBJECT;
        }
        JSONObject jsonObject = JSON.parseObject(feature);
        jsonObject.put(key, val);
        String value = JSON.toJSONString(jsonObject);
        ReflectUtil.setFieldValue(obj, FIELD_FEATURE, value);
        return obj;
    }

    public static <T> T getObject(String feature, String key, Class<T> clazz) {
        return JSON.parseObject(feature).getObject(key, clazz);
    }

    public static String getString(String feature, String key) {
        return JSON.parseObject(feature).getString(key);
    }

    public static long getLong(String feature, String key) {
        return JSON.parseObject(feature).getLongValue(key);
    }

    public static int getInteger(String feature, String key) {
        return JSON.parseObject(feature).getIntValue(key);
    }

    public static boolean getBoolean(String feature, String key) {
        return JSON.parseObject(feature).getBooleanValue(key);
    }
}
```

### @FeatureToJson

entity feature(String) to dto feature(Map)

```java
import org.mapstruct.Qualifier;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Qualifier
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface FeatureToJson {
}
```

### @FeatureToMap

dto feature(Map) to entity feature(String)

```java
import org.mapstruct.Qualifier;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Qualifier
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface FeatureToMap {
}
```

### OrderMapper

可配置某些属性的转换策略, 需要通过 uses 属性引入依赖的具体转换实现类。 

```java
@Mapper(componentModel = "spring", uses = {FeatureUtil.class})
public interface OrderMapper extends EntityMapper<OrderDTO, Order> {

    /**
     * feature 字段交由 @FeatureToJson 注解修饰的方法去处理
     */
    @Mapping(target = "feature", qualifiedBy = FeatureToJson.class)
    @Override
    Order toEntity(OrderDTO dto);

    /**
     * feature 字段交由 @FeatureToMap 注解修饰的方法去处理
     */
    @Mapping(target = "feature", qualifiedBy = FeatureToMap.class)
    @Override
    OrderDTO toDto(Order entity);

    default Order fromId(Long id) {
        if (id == null) {
            return null;
        }
        Order order = new Order();
        order.setId(id);
        return order;
    }
}
```

不管有多少个模型, 只需要加注解即可

- @Mapping(componentModel = "spring", uses = {FeatureUtil.class})
- @Mapping(target = "feature", qualifiedBy = FeatureToJson.class)
- @Mapping(target = "feature", qualifiedBy = FeatureToMap.class)
