# BookingInfoProcessItem

## 餐食预处理

1、 为什么价格日历会有两个地方？

```undefined
List<QDailyPrice> qDailyPrices = isOriginal ? qOrderProduct.getProduct().getOriginalDailyPrices() : qOrderProduct.getProduct().getDailyPrices();
```

## Bookinginfo

* 设置邮箱
* 设置早餐
* 最大可订数量

  * 其中预售会特殊处理一下。如果是预售券是连住，则最大可订间数是预售券的可用数量；如果非连住，则最大间数等于可用数量/晚数
  * 如果预售没有特殊处理，则取

‍

‍
