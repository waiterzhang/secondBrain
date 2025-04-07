# DatePriceBookingItem

1. 将报价的数据结构QDailyPrice转换到内部的数据结构DailyPrice。量者数据结构整体差不多。主要是餐食情况有一点特殊处理：增加了一些表达餐食信息的文案  
    这个计算没看懂：  
    ​![image](image-20240514100554-bevv8wg.png)​
2. 将DailyPrice模型，转化为DatePrice模型。（这个模型看起来更可用了一些）

    * 可订房间数量 = 价格非空 and 房间数大于0 and 房态有效
    * 价格计算没看懂  
      ​![image](image-20240514101609-7d60yap.png)​
3. 将DailyPrice模型，转化为MobDatePrice模型，相较于上面，有一些变化（看起来信息更多了），这个模型被放在`datePrice`​字段了

    * 增加了餐食数量的信息；
    * 房态信息
    * 增加了一些智行用到的优惠：低价、减优惠

上面一共四个模型：QDailyPrice、DailyPrice、DatePrice、MobDatePrice；

* 其中QDailyPrice是报价的数据模型，DailyPrice是booking根据报价模型进行初步转换的模型，两者基本一致；
* DatePrice、MobDatePrice均为DailyPrice转换而来，增加了一些信息、例如房态，价格；
* 其中DatePrice被放在了`getBookInfoObj().setDatePrice(datePrices)`​节点下面。该节点为下单，给到前端之后，前端需要原样回传，不允许修改。

‍
