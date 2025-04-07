# O页扣款路径-简要版

## 支付方式文案及处理

1. 信用住+先住后付，前端统一展示为“先住后付”

    * 信用住or先住后付判断依据：mainOrder.orderControlInfo.orderCreditType,一共4种取值：

      * flash_credit 闪住
      * pay_later 先住后付
      * credit_guarantee 信用担保
      * null  
        如果为null，做老版本兼容去从以下路径进行匹配：

        * 闪住判断依据：mainOrder.orderControlInfo.flashCreditType=POST_AUTH闪住后授权 或者 orderProduct.productTracking.trackingParameters.zhiMaCredit
        * 先住后付：mainOrder.orderExtension.trackData.isPayLater
        * 信用担保：mainOrder.orderExtension.trackData.creditGuarantee
    * rule构造

      * 如果订单=CHECKED_OUT 且  有杂费【取值路径：orderData.debtOrder.orderPayRecords.amount（type = ZHIMA_MISC ）】 且 无房费【取值路径：orderData.debtOrder.orderPayRecords.amount（type=ROOM_FEE_DEBT）】：

        * **您在房间内产生【杂费金额】费用未支付，请尽快还款**
      * 如果含有杂费押金：  
        忽略，现在已经没有含押闪住了
      * 否则：获取还款单【取值路径：orderData.repayOrder】，查看还款状态【此处会对还款状态做一定处理】

        * 如果是闪住欠款PARTNER_DEDUCT状态：

          * 展示：**欠款 %s**（qunar垫付的，也就是用户需要还款的金额）
        * 如果父单parentOrder不为空 且 履约状态（parent.tradeAuth.fulfillment）为扣款失败PAY_FAIL

          * 展示：**欠款 %s**（履约总金额或者待履约金额，金额=parent.tradeAuth.fulfillment.getFulfillFees.getAmount加总）
        * 否则

          * 如果订单详情状态OrderDetailStatusInfo为待授权且为先住后付PayLater，则展示配置文案，默认值：**授权时无需支付房费，离店后再从您授权的账户中扣款**
          * 如果订单详情状态OrderDetailStatusInfo为待授权且为闪住isFlashCredit，则展示配置文案，默认值：**授权时无需支付房费，离店后再从您授权的账户中扣款**
          * 如果订单状态orderData.orderBase.orderStatus为CANCELLED:

            * 芝麻分：：

              * 如果金额大于0，rule=**取消订单将自动扣款¥%s**
            * 非芝麻分：

              * 如果金额大于0，rule = **取消订单将自动扣款¥%s**
            * 如果上述金额为0，则展示：**订单已取消，不再扣取订单费用**
          * 如果是先住后付isPayLater 或者 闪住isFlashCredit

            * 如果杂费金额>0:

              * 如果是闪住：文案从配置，默认值：**扣款于离店日%s开始发起，将包含您在房间内的消费¥%s**
              * 非闪住：文案从配置中取，默认值：**扣款于离店日%s10:00后发起，将包含您在房间内的消费¥%s**
            * 如果杂费金额=0，无杂费：

              * 如果是闪住：文案从配置中取，默认值：**扣款于离店日%s开始发起**
              * 非闪住：**扣款于离店日%s10:00后发起**
          * 都未匹配展示：**离店后再从您的授权账户中扣款**

          ‍

2. 分期

    * 分期的判断依据路径【路径：mainOrder.orderControlInfo.installmentOrder】
    * 无rule
3. 预付

    * 预付判断依据【路径：mainOrder.orderPayInfo.payType=ONLINE】
    * 无rule
4. 现付担保

    * 现付担保判断依据【路径：mainOrder.orderPayInfo.payType = CASH 且 orderProduct.orderProductControlInfo.needGuarantee = true】
    * 如果总金额 > 0：

      * 构造rule

        * 如果是信用担保类型：

          * 如果订单详情状态OrderDetailStatusInfo为待授权且为先住后付isPayLater，则根据配置展示，默认：**授权时无需支付房费，离店后再从您授权的账户中扣款**
          * 如果订单详情状态OrderDetailStatusInfo为待授权且为闪住isFlashCredit，则根据配置展示，默认：**授权时无需支付房费，离店后再从您授权的账户中扣款**
          * 如果订单状态orderData.orderBase.orderStatus为CANCELLED:

            * 芝麻分：：

              * 展示：**取消订单将自动扣款¥%s**
            * 非芝麻分：

              * 展示 ：**取消订单将自动扣款¥%s【与芝麻分金额计算方式不一样】**
            * 如果金额为0，则展示：**订单已取消，担保自动撤销**
        * 如果是先住后付isPayLater 或者 闪住isFlashCredit

          * 如果杂费金额>0:

            * 如果是闪住：文案从配置中取，默认：**扣款于离店日%s开始发起，将包含您在房间内的消费¥%s**
            * 非闪住：文案从配置中取，默认：**扣款于离店日%s10:00后发起，将包含您在房间内的消费¥%s**
          * 如果杂费金额=0，无杂费：

            * 如果是闪住：文案从配置中取，默认：**扣款于离店日%s开始发起**
            * 非闪住：**扣款于离店日%s10:00后发起**
          * 上述均不匹配展示：**授权后可享受平台授信担保，房间整晚保留**
        * 非信用担保类型：

          * 订单状态已授权 或 已支付展示：**担保金%s用于保留房间，离店后自动原路退**
          * 其它状态展示：**担保后房间整晚保留，离店后担保金自动原路退**
5. 现付非担保

    * text = 到店付
    * rule不展示

## 扣款规则文案及处理

所用字段progressInfo.tips

* 有开关控制
* 信用住、先住后付、信用担保现在都没有退款进度信息【以前有】
* 根据订单号查询交易，获取展示数据，命中以下条件则展示：

  * 依据查询交易数据`ShowResponse.display为true`​​:

    * 如果订单状态为`CANCELLED`​​，则展示`订单取消，支付金额将原路退回`​​，
    * 否则，展示`担保金将退回您的账户`​​
  * 如果订单状态 = `CHECKED_OUT`​​  
    有杂费欠款，没有房费欠款时展示

    * title = `扣款失败`​​
    * tip = `房间内消费扣款失败`​​
  * 如果可申请返现，判断依据订单状态!=`CANCELLED`​​且!=`NO_SHOW`​​ 且!= `REJECTED`​​且 返现金额>0。

    * title = `可返现`​​
    * tip = `离店后30天内，可领返现%s%s`​​返现金额 + `（含已兑换券）`​​如果兑换过返现商品  
      【兑换返现商品的依据路径：`orderData.mainOrder.orderExtension.secondStatus.cashRedeemApply.applyStatus`​​】
  * 如果是返现单：

    * 返现状态：`orderData.getMainPay().cashPay.getCashStatus`​​ =`CASH_BACK_SUCCESS`​​返现成功

      * title = `返现成功`​​
      * tip =`可从您的去哪儿余额帐户查询`​​
    * 返现状态：`orderData.getMainPay().cashPay.getCashStatus`​​ =`CASH_BACK`​​待返现

      * title = `返现中`​​
      * tip = `预计x月y日发放到去哪儿余额帐户（以实际返现为准）`​​
    * 返现状态：`orderData.getMainPay().cashPay.getCashStatus`​​ =`START`​​待返现

      * title = `返现中`​​
      * tip = `预计x月y日发放到去哪儿余额帐户（以实际返现为准）`​​
      * 上下两个不同的待返现，区别在于金额计算有所区别

‍

### 取值路径

1. 信用住&先住后付的判断取值路径：  
    ​`mainOrder.orderControlInfo.orderCreditType`​,一共有四种取值：

    * ​`**flash_credit**`​**闪住【命中信用住+闪住】**
    * ​`**pay_later**`​ ** 先住后付【命中先住后付】**
    * ​`credit_guarantee`​ 信用担保
    * null

    如果为null，做一下老版本兼容：

    * 闪住判断依据：`mainOrder.orderControlInfo.flashCreditType`​=`POST_AUTH`​闪住后授权 **或者 **​`orderProduct.productTracking.trackingParameters.zhiMaCredit`​
    * 先住后付：`mainOrder.orderExtension.trackData.isPayLater`​
    * 信用担保：`mainOrder.orderExtension.trackData.creditGuarantee`​
2. 分期的判断依据：

    ​`mainOrder.orderControlInfo.installmentOrder`​
3. 预付的判断依据：  
    ​`mainOrder.orderPayInfo.payType`​=`ONLINE`​
4. 现付担保的判断依据：  
    ​`mainOrder.orderPayInfo.payType`​ = `CASH`​ 且 `orderProduct.orderProductControlInfo.needGuarantee = true`​

‍

‍
