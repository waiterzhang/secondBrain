# HotelPriceBolt 查报价

查询报价的参数

* 促销优惠来自于四个方面：

  * submitReq里面的discounts，适配的过程中会做一些校验。
  * 权益云的权益会被合并到PromotionInfo里面。未选中的权益会被过滤掉。
  * submitReq里面的`specialPromotion`​
  * 恢复填单页不展示的优惠，如分享免单等。不可见的优惠放在`extraParams`​里面，key是`specialPromotionActivityInfo`​

    > ​`PromotionInfo`​里面有一个`List<BizPromotionInfo> allPromotionInfos`​字段，但是看起来这个List并非真实一个List，看了很多地方，都是一个List里面只有一个元素，元素的类型是`BizPromotionInfo`​。
    >

  促销里面的优惠被整合之后，似乎只拿了`promotionInfo.getAllPromotionInfos().get(0).getAdditionInfos()`​里面的ticketPromtionId给到报价。
* 会员身份
* 版控：

  * 透传Venus的版控
  * 大量的各种需求的版控。看起来非前端的版控是不涉及bizVersion的。

‍

**使用**

‍
