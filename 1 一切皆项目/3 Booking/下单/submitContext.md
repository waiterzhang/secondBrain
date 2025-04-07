# submitContext

* ​`addFuyingRentCarCoupon`​
* 填充SubmitRequest

  * 根据版本信息移除一些参数；
  * 将`request.weakTips`​里面取消规则塞到`request.cancellation`​
  * 处理用户输入的"其它要求"中的特殊字符的替换

  * 适配入住人信息：

    * 过滤无效入住人信息（空格等）
    * 如果入住人数量大于房间数量，则取房间数量的入住人
    * 老版idcard（身份证）兼容
    * 适配联系人：将第一个入住人的名称适配到联系人上。
  * 适配优惠参数：

    * 将下单带过来的优惠request.disocuts里面的内容提取出来。计算返现+直减的总优惠放到`SubmitRequest.totalPrize`​。红包、签到多返不算在内，但是会记录下来。（红包估计已经没了）
* 适配​`SubmitContext.extraParams`​参数：

  * 将`submitRequest.bookInfo.extraParam`​、`submitRequest.extra`​里面的额外参数都放到`SubmitContext.extraParams`​
* 将请求中带过来的待支付金额`submitRequest.totalPrice`​放到`SubmitContext.originTotalPrice`​
* 适配`SubmitContext.submitRequestExt`​参数：

  * 将联系人手机号`SubmitRequest.contactPhone`​解密后的手机号，以及担保金额`submitRequest.totalVouchMoney`​都放到`SubmitContext.submitRequestExt`​
* 创建订单信息​`OrderInfo`​：

  * 入离日期、房间数、到店时间、离店时间（默认晚上六点）
  * 设置钟点房的入离时间（将入离时间转换为易懂文案）
  * 设置房型相关信息：产品房型id、wrapperId，来自于`submitRequest.bookInfo.roomId/wrapperId`​
  * 写入请求的来源​`bookingChannel`​：Pc/移动端
  * 用户的特殊要求

    * ​`remark`​=`submitRequest.otherRequire`​，这个会过滤emoji
    * ​`remarkCodes`​=​`submitRequest.specialRemarkCodes`​
    * ​`specialRemark`​=`submitRequest.specialRemarkExtra`​，用户手动输入的特殊要求  
      上面的内容就是填单页的特殊要求，特殊要求有很多种呈现形式：手动输入、选择，这些选项的露出和供应商有关系。
  * 记录联系人姓名、解密后手机号、邮箱、百度id、用户级别、绑卡类型，这些信息基本来自于`submitRequest.bookInfo`​
  * 设置返现`cashBack`​，返现只支持支付类型PtType = 2或0或4、且优惠金额`submitRequest.totalPrize`​非空
  * 设置客户端平台，根据vid；
  * 设置支付类型、经纬度、bookingsource=cid、闪住、授权渠道列表等。
* 处理优惠：

  * 处理请求discounts里面的优惠：

    * 校验优惠是否有效：优惠名称、`preferId`​、闪住是否有闪住标识、名称是否已`promotion_offer`​开始；
    * 将下单带过来的`DiscountForBook`​结构，转化为`PromotionInfo`​结构
  * 处理所有优惠，补全用户看不见的促销以及权益云信息：

    * 恢复填单页不展示的特殊优惠，如：撒币、分享免单；这些不可见的优惠来源于`SubmitContext.extraParams`​，key = `specialPromotionActivityInfo`​；
    * 恢复的内容包括金额类优惠（恢复时默认填写1张）和非金额类优惠。
    * 权益云：权益云的源数据来自于`submitRequest.rightBooks`​，目标转换的结构`PromotionInfo`​。注意`BizPromotionInfo.amount`​似乎并不特质优惠金额，也可能是选中的份数，根据是否是金额类权益区分。
  * 特殊优惠`SubmitRequest.specialPromotion`​
  * 将上述三部分优惠聚合在一起；
* 优惠校验：

  * 联系人手机号、入离日期、到店时间、预定房间数量等等
* 计算支付详情，放到`SubmitContext.submitPriceInfoDetail`​：

  * ​`payTotalPrice`​：总支付价
  * ​`hotelTotalPayPrice`​：酒店侧总支付价
  * ​`excludeCashbackHotelTotalPayPrice`​：酒店侧总支付价减去返现
  * ​`cashBackPrice`​：返现总价
  * ​`fuyingPrice`​：辅营商品总价
* 构建`SubmitContext.orderRequest`​,包括：

  * ​`ProductInfo`​产品信息:`wrapperId`​、`productId`​、`productRoomName`​、支付相关信息、产品唯一标识、房型信息`RoomInfo`​、是否团队房等；
  * ​`orderInfo`​订单信息
  * c参
  * 携程设备指纹
  * trace
  * ​`SubmitRequest`​
  * ​`extraParams`​
* 处理取消无忧商品`submitRequest.fuying.products`​：

  * 可售间数校验
  * 支付金额配置
* 设置B参、cookie、UUID和一些标记

‍
