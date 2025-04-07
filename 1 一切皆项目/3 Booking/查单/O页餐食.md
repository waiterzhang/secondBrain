# O页餐食

O页餐食中展示权益云餐食的判断条件，

前端的展示条件：

* noMeal = true  && pointsRight中有promotionCode=FREE_BREAKFAST的节点时则展示权益云早餐

noMeal = true 的条件：

* 取值来源：订单报价数据​`qOrderProduct.getProduct().getDailyPrices()`​
* 判断条件：入住第一天无餐食（不区分固定or可选餐食） **且 ** 多天餐食数据存在不一致（一致指的是任意两天早、中、晚、可选类型、可选数量均相同）

promotionCode=FREE_BREAKFAST：

* 取值来源权益云：`orderData.getMainOrder().getOrderProduct().getOrderProductBase().getPromoteInfo()`​中的`promotionCode`​为`FREE_BREAKFAST`​的权益
* 判断条件：该权益多天总量不为0，则该节点不为空
