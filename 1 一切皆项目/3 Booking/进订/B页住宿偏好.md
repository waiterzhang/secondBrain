# B页住宿偏好

**前端展示的判断逻辑：**

const showRequire = needOtherRequire || specialRequireOpts.length > 0 || checkFields.length > 0;

> needOtherRequire 有值  且非false 非0   就会判断对真

展示的内容为：specialRequireOpts  
​![image](image-20231218205353-8cd9epo.png)​  

**后端塞数据的逻辑：**

> ​`specialRemarks`​的填充逻辑：
>
> ​`取值：qOrderProduct.getProduct().getRatePlan().getBookingRule().getSupportRequirements()`​如果不为空，则直接填充；
>
> 否则：如果该代理商在不使用qta包装的房型等特殊要求的wrapper的配置中则不填充任何内容；
>
> 否则：填充窗户相关的内容，取值：`qOrderProduct.getRoom().getPropInfo().getWindowProperty()`​如果是报价中是`部分无窗`​或者`不确定`​则添加为`有窗`​；填充`尽量无烟`​选项

* needOtherRequire:

  * 判断携程的输入项是否有其它要求，判断依据：如果携程不支持，则为“false”；`qOrderProduct.getProduct().getRatePlan().getBookingRule().isCustomRequire()`​字段

  *  版本不支持（版控 + 端支持 ）则为“false”；
  * 否则为“true”
* specialRequireOpts填充数据：

  * 床型要求：

    * 如果是c或者hecheng接口则不填充
    * 如果在不使用qta包装的房型等特殊要求的wrapper配置中，则不填充
    * 填充床型相关规则，取值`qOrderProduct.getRoom().getPropInfo().getBedType()`​== `BIG_DOUBLE`​（`大床或双床`​）则填充`无床型偏好`​、`尽量安排大床`​、`尽量安排双床`​
  * 抽烟要求：

    * 如果报价中包含“尽量无烟”的规则。取值依据：​`specialRemarks`​，则塞入`尽量安排无烟房`​、`尽量安排吸烟房`​这两个选项。
  * 房间要求：

    * 如果报价中包含`有窗`​、`尽量高层`​、`尽量安排安静房间`​、`尽量安排蜜月布置`​的规则，则上述内容与报价取交集。报价取值逻辑：`specialRemarks`​
    * 填充`尽量无烟`​选项
* checkFields（复选框）：

  * 填充房型需求相关的复选框，填充数据来源​`specialRemarks`​

‍
