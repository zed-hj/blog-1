---
title: 使用策略模式代替if
date: 2019/4/30 21:14:00

categories: 
  - divide and rule
---

**业务场景:**

采购仓储,用户到1688下单,系统要监听处理1688推过来的订单状态消息,例如商品改价、订单付款、订单发货、确定收货...

每个状态都有不同的处理逻辑,最简单粗暴的做法当然是直接if else了,但是!!! 这不优雅!!!

<!--more-->

## IF ELSE

### AlibabaMessageTypeEnum(1688消息类型枚举)

```java
public enum AlibabaMessageTypeEnum {
    PRODUCT_PRODUCT_CROSSBOARD_INFORM("PRODUCT_PRODUCT_CROSSBOARD_INFORM", "一键铺货"),
    ORDER_BUYER_VIEW_ORDER_PRICE_MODIFY("ORDER_BUYER_VIEW_ORDER_PRICE_MODIFY", "商品改价"),
    ORDER_BUYER_VIEW_ORDER_PAY("ORDER_BUYER_VIEW_ORDER_PAY", "订单付款"),
    ORDER_BUYER_VIEW_ANNOUNCE_SENDGOODS("ORDER_BUYER_VIEW_ANNOUNCE_SENDGOODS", "订单发货"),
    ORDER_BUYER_VIEW_PART_PART_SENDGOODS("ORDER_BUYER_VIEW_PART_PART_SENDGOODS", "部分发货"),
    ORDER_BUYER_VIEW_ORDER_COMFIRM_RECEIVEGOODS("ORDER_BUYER_VIEW_ORDER_COMFIRM_RECEIVEGOODS", "确定收货"),
    ORDER_BUYER_VIEW_ORDER_SUCCESS("ORDER_BUYER_VIEW_ORDER_SUCCESS", "交易成功"),
    ORDER_BUYER_VIEW_ORDER_BUYER_CLOSE("ORDER_BUYER_VIEW_ORDER_BUYER_CLOSE", "买家关闭订单"),
    ORDER_BUYER_VIEW_ORDER_SELLER_CLOSE("ORDER_BUYER_VIEW_ORDER_SELLER_CLOSE", "卖家关闭订单");

    private String code;
    private String msg;

    AlibabaMessageTypeEnum(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    // getter and setting...
}

```

### AlibabaWebSocketMsgProcessService(监听1688 WebSocket消息)
```java
@Override
public boolean onMessage(WebSocketMessage message) throws MessageProcessException {
    try {
        String messageType = JSON.parseObject(message.getContent()).getString("type"));
        if (AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_PAY.getCode().equals(messageType)) {
            // 订单付款逻辑
        } else if (AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ANNOUNCE_SENDGOODS.getCode().equals(messageType)) {
            // 订单发货逻辑
        } else if (AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_COMFIRM_RECEIVEGOODS.getCode().equals(messageType)) {
            // 确定收获逻辑
        }
        // ...
    } catch (Exception e) {
        log.error("[处理1688 WebSocket 消息异常]", e);
    }
    return true;
}
```
弊端:

- 强耦合 不优雅
- 每新监听一种消息的时候都要多一个 `if` `else` 和相应的逻辑
- 消息类型多了 `if` `else` 太多逻辑不容易理解

## 策略模式

**思路:**

将处理逻辑分散到不同的类

1. 提炼出一个策略接口
2. 根据消息类型实现不同的消息处理逻辑

### AlibabaMessageStrategy(1688消息处理策略)

```java
public interface AlibabaMessageStrategy {

    /**
     * 实现该方法处理不同的消息
     */
    void handle(WebSocketMessage message, String key);
}
```
### AlibabaMessageHandle(根据消息执行相应的逻辑)

```java
public class AlibabaMessageHandle {

    private Map<String, AlibabaMessageStrategy> strategyMap = new HashMap<>();

    /**
     * 使用HashMap实现根据key获取具体的消息实现类
     *
     * 为什么要在Spring容器加载后加载策略?
     *  这是因为消息处理函数内实际依赖了其他注入进来的Service(这里省略了),
     *  如果在static代码块内或构造方法内获取策略实现会出现service为null的情况,类似js的闭包.
     */
    @PostConstruct
    public void init() {
        strategyMap.put(AlibabaMessageTypeEnum.PRODUCT_PRODUCT_CROSSBOARD_INFORM.getCode(), productProductCrossboardInform());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_PRICE_MODIFY.getCode(), orderBuyerViewOrderPriceModify());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_PAY.getCode(), orderBuyerViewOrderPay());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ANNOUNCE_SENDGOODS.getCode(), orderBuyerViewAnnounceSendgoods());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_PART_PART_SENDGOODS.getCode(), orderBuyerViewPartPartSendgoods());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_COMFIRM_RECEIVEGOODS.getCode(), orderBuyerViewOrderComfirmReceivegoods());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_SUCCESS.getCode(), orderBuyerViewOrderSuccess());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_BUYER_CLOSE.getCode(), orderBuyerViewOrderBuyerClose());
        strategyMap.put(AlibabaMessageTypeEnum.ORDER_BUYER_VIEW_ORDER_SELLER_CLOSE.getCode(), orderBuyerViewOrderSellerClose());
    }

    /**
     * 根据消息类型执行相应的处理实现
     */
    public void exec(String type, WebSocketMessage message, String key) {
        AlibabaMessageStrategy alibabaMessageStrategy = strategyMap.get(type);
        if (null != alibabaMessageStrategy) {
            alibabaMessageStrategy.handle(message, key);
        }
    }

    /**
     * 一键铺货
     */
    private AlibabaMessageStrategy productProductCrossboardInform() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [一键铺货] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 商品改价
     */
    private AlibabaMessageStrategy orderBuyerViewOrderPriceModify() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [商品改价] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 订单付款
     */
    private AlibabaMessageStrategy orderBuyerViewOrderPay() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [订单付款] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 订单发货
     */
    private AlibabaMessageStrategy orderBuyerViewAnnounceSendgoods() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [订单发货] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 部分发货
     */
    private AlibabaMessageStrategy orderBuyerViewPartPartSendgoods() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [部分发货] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 确定收货
     */
    private AlibabaMessageStrategy orderBuyerViewOrderComfirmReceivegoods() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [确定收货] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 交易完成
     */
    private AlibabaMessageStrategy orderBuyerViewOrderSuccess() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [交易完成] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 买家关闭订单
     */
    private AlibabaMessageStrategy orderBuyerViewOrderBuyerClose() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [买家关闭订单] : {}", message.getId());

            // 处理逻辑
        };
    }

    /**
     * 卖家关闭订单
     */
    private AlibabaMessageStrategy orderBuyerViewOrderSellerClose() {
        return (message, key) -> {
            StaticLog.info("[消费1688 WebSocket消息] - [卖家关闭订单] : {}", message.getId());

            // 处理逻辑
        };
    }
}
```

使用策略模式将各种类型的消息处理逻辑单独拎出来了,这样我们可以很清晰的关注某个处理类.

### AlibabaWebSocketMsgProcessService(策略模式后的消息监听)

```java
@Override
public boolean onMessage(WebSocketMessage message) throws MessageProcessException {
    try {
        alibabaMessageHandle.exec(JSON.parseObject(message.getContent()).getString("type"), message, key);
    } catch (Exception e) {
        log.error("[处理1688消息异常]", e);
    }
    return true;
}
```

## 结尾

1. 抽象一个策略接口
2. 用 `Map` 存储实现类(类似工厂模式)

这样我们就能通过策略的 `key` 获取到对应的实现类,从而执行不同的处理逻辑,并且关注点从一堆 `if` `else` 转换到了很清晰的某个实现类.