# O页扣款路径-详细版

### 信用住&先住后付

1. 判断是否是信用住&先住后付

    * 判断依据：`mainOrder`​.`orderControlInfo`​.`orderCreditType`​,一共4种取值：

      * ​`**flash_credit**`​**闪住【命中信用住+闪住】**
      * ​`**pay_later**`​ ** 先住后付【命中先住后付】**
      * ​`credit_guarantee`​ 信用担保
      * null，如果为null，做一下老版本兼容：

        * 闪住判断依据：`mainOrder`​.`orderControlInfo.flashCreditType`​=`POST_AUTH`​闪住后授权 **或者 **​`orderProduct.productTracking.trackingParameters.zhiMaCredit`​
        * 先住后付：`mainOrder.orderExtension.trackData.isPayLater`​
        * 信用担保：`mainOrder.orderExtension.trackData.creditGuarantee`​
2. 费用计算

    * 获取到芝麻分的杂费还款金额，目前只有[信用住芝麻分]会有杂费的还款单；`orderData`​.`debtOrder`​.`orderPayRecords`​.`amount`​（`type = ZHIMA_MISC`​ ）
    * 获取到用户的房费欠款金额；`orderData`​.`debtOrder`​.`orderPayRecords`​.`amount`​（type=`ROOM_FEE_DEBT`​）
    * 根据信用类型（`flash_credit`​ or `pay_later`​）获取配置文件中的支付图标tips，将`flash_credit`​和`pay_later`​统一展示为”先住后付“；

      ‍
    * 如果订单=`CHECKED_OUT`​ 且 （有杂费 或者 无房费）待付款：组装

      * title = 先住后付
      * 金额
      * desc=待还款
      * rule=`您在房间内产生%s费用未支付，请尽快还款`​（杂费金额）
      * poptips=先住后付tips
    * 如果含有杂费押金（`miscOrder`​.`depositPayAmount`​>0)，含押闪住。组装：

      * title = 先住后付
      * 金额
      * rule=含押金x元
    * 否则：

      * 获取还款单：`orderData`​​.`repayOrder`​​，并查看还款状态：

        * 如果还款单状态为取消​`CANCELED`​​ 或者 还款单金额为0（`repayOrder`​​.`totalMoney`​​）,则还款单状态为​`CANCELED`​​
        * 如果还款单状态为`PAID`​​，则还款单根据条件为`REPAY_SUCCESS`​​或`DEDUCT_SUCCESS`​​（均展示为`扣款成功`​​）
        * 如果还款单状态为部分支付成功`PARTPAID`​​，则还款单状态为​闪住欠款`PARTNER_DEDUCT`​​
        * 如果还款单状态为等待支付`UNPAY`​​，则还款单为等待扣款​`WAIT_DEDUCT`​​
        * 兜底状态：`CANCELED`​​
      * 如果是闪住欠款`PARTNER_DEDUCT`​​状态

        * 组装，其中rule=`欠款 %s`​​（qunar垫付的，也就是用户需要还款的金额）
      * 如果父单`parentOrder`​​不为空 且 履约状态（`parent.tradeAuth.fulfillment`​​）为扣款失败`PAY_FAIL`​​

        * 组装，其中rule=`欠款 %s`​​（履约总金额或者待履约金额，金额=`parent.tradeAuth.fulfillment.getFulfillFees.getAmount`​​加总）
      * 否则：

        * 处理`unPayDesc`​​

          * 如果当前订单处于追加支付状态`mainOrder`​​.`orderPayInfo`​​.`additionalPayment`​​.`status`​​=`INIT`​​则无担保描述
          * 如果是(程信分授权 或者 后付授权类型） 且 信用支付类型单的授权记录状态`parentOrder.tradeAuth.auth.authStatus`​​ =`INIT`​​则展示​`未授权`​​
          * 否则：

            * 如果订单状态为未支付

              * 芝麻分，则展示​`未授权`​​
              * 如果`hasPackage`​​，则展示​`未支付`​​
              * 否则展示：​`未担保`​​
          * 兜底，不展示
        * 处理`闪住、先住后付、信用担保规则文案`​​rule

          * 当存在扣款进度时`OrderDetailProgressInfo.items=creditDeduct`​​​，费用明细处的信用类型文案不展示
          * 如果订单详情状态​`OrderDetailStatusInfo`​​​为`待授权`​​​且为先住后付`isPayLater`​​​，则展示配置文案（key=`PAY_LATER_AUTHROZING_FEE_DESC`​​​），兜底：`授权时无需支付房费，离店后再从您授权的账户中扣款`​​​
          * 如果订单详情状态​`OrderDetailStatusInfo`​​​为`待授权`​​​且为闪住`isFlashCredit`​​​，则展示配置文案（key=`FLASH_CREDIT_AUTHROZING_FEE_DESC`​​​），兜底：`授权时无需支付房费，离店后再从您授权的账户中扣款`​​​
          * 如果订单状态`orderData.orderBase.orderStatus`​​​为`CANCELLED`​​​:

            * 芝麻分：：

              * 金额 `orderData.fundOrder`​​​ + 汇率 `orderData.orderProduct.currencyExchange`​​​这两方面组合计算出来的用户实际支付的所有费用，包括但不限于：房费，组合支付费用，保险费用，快递费用
              * 如果金额大于0，rule=​`取消订单将自动扣款¥%s`​​​（上述金额）
            * 否则：

              * 如果还款单`repayOrder`​​​不为null：还款金额来自于支付流水`repayOrder.payTraces.get("REPAY").amount`​​​
              * 否则来自于待履约金额`parent.tradeAuth.fulfillment.fulfillFees.amount加总`​​​
              * 如果金额大于0，rule = `取消订单将自动扣款¥%s`​​​（上述金额）
            * 如果上述金额为0，则展示​`订单已取消，不再扣取订单费用`​​
          * 如果是先住后付`isPayLater`​​ 或者 闪住​`isFlashCredit`​​

            * 获取杂费金额：

              如果是后付授权（`MainOrder.OrderControlInfo.OrderAuthInfo.authPlatform.AuthType`​​=`TRADE_AUTH`​​）则取履约（`fulfillFees`​​）否则取`orderData.miscFeeOrder`​​
            * 如果杂费金额>0:

              * 如果是闪住：文案从配置中取，兜底：`扣款于离店日%s开始发起，将包含您在房间内的消费¥%s`​，金额=杂费金额
              * 否则：文案从配置中取，兜底：`扣款于离店日%s10:00后发起，将包含您在房间内的消费¥%s`​，金额=杂费金额
            * 如果杂费金额=0，无杂费：

              * 如果是闪住：文案从配置中取，兜底：`扣款于离店日%s开始发起`​
              * 非闪住：​`扣款于离店日%s10:00后发起`​
          * 否则：​`离店后再从您的授权账户中扣款`​
        * 根据上述参数组装，

### 分期

1. 判断依据：  
    ​`mainOrder.orderControlInfo.installmentOrder`​
2. 数据组装

	title = `订单金额`​

	金额 =`currencyTotalPrice`​

	popTips=配置文件中取key=​`installment`​

### 预付

1. 判断依据：  
    ​`mainOrder.orderPayInfo.payType`​=​`ONLINE`​
2. 数据组装

    * 判断是否支付`orderData.mainPay.orderPay.status`​=​`PAY`​
    * title = ​`在线付`​
    * ​`price`​ = ​`currencyTotalPrice`​
    *  desc = 未支付则展示未支付，否则不展示
    * rule = null
    * poptips=从配置中拿，key=​`online`​
    * ​`showButton`​ = isNotPay（未支付，则showButton为true）
    * ​`actId`​=​`PAY.actId`​

### 现付担保

1. 判断依据  
    ​`mainOrder`​​.`orderPayInfo.payType`​​ = `CASH`​​ 且 `orderProduct.orderProductControlInfo.needGuarantee = true`​​
2. 数据组装  

    * 是否是信用担保（免担保）订单

      ​`mainOrder.orderControlInfo.orderCreditType`​​  
      兼容老订单,当`orderCreditType`​​ = null的时候，看这个字段：  
      ​`mainOrder.orderExtension.trackData.get("creditGuarantee")`​​ =true
    * 是否需要展示免担保或者在线担保，只有需要授权或支付时需要展示。判断条件订单详情状态`OrderDetailStatusInfo.statusEnum = `​​(`AUTHORIZING`​​ 或 `GUARANTEEING`​​ 或 ​`PAYING`​​)
    * ​`popupTips`​​ = 从配置`AppXmlContext.getAppXmlConfig().getPaymentIconPopupTips()`​​中取key，key 的选择：

      * 如果是信用担保，取​`credit_guarantee`​​
      * 非信用担保，且wrapper在配置中（`isBookingCom`​​没看明白，大概意思是确定某个报价中的wrapper是否在配置中，黑白名单？），取`guarantee_alt`​​，否则取`guarantee`​​
    * 获取搭售金额。来源：`PackageMobOrderDetail.totalPayMoney`​​
    * 总金额 = 

      * 如果有追加支付：用户已经支付成功的金额（如果有，来源`orderData.mainPay.orderPay.orderPayRecordList`​​） +搭售金额
      * 如果没有追加支付：担保金额`guaraPrice`​​，担保金额来源（`orderData.getFundOrder`​​ +`orderData.getOrderProduct().getCurrencyExchange()`​​） + 搭售
    * 如果总金额>0：

      * text =  
        如果是信用担保`credit_guarantee`​​​，则展示`免担保金`​​​  
        否则：

        * package = true(为true条件：`OrderDetailResult.insuranceView/couponView/qstarView任意非空`​​​)，展示`在线支付`​​​
        * 否则：展示​`在线担保`​​​
      * ​`unPayDesc`​​​ =

        * 非追加支付状态，不展示
        * 如果（是程信分 或 后付授权类型）

          * 如果授权状态`orderData.parentOrder.tradeAuth.authStatus = AuthStatus.INIT`​​​则展示`未授权`​​​
          * 否则不展示。
        * 否则：

          * 订单状态为`待授权/支付`​​​，则展示：`未授权`​​​
          * ​`hasPackage`​​​ = true，则展示：​`未支付`​​​
          * 否则展示​`未担保`​​​
      * rule =

        * 如果是信用担保类型：

          * 当存在扣款进度时`OrderDetailProgressInfo.items=creditDeduct`​​​，费用明细处的信用类型文案不展示
          * 如果订单详情状态`OrderDetailStatusInfo`​​​为`待授权`​​​且为先住后付`isPayLater`​​​，则展示配置文案（key=`PAY_LATER_AUTHROZING_FEE_DESC`​​​），兜底：`授权时无需支付房费，离店后再从您授权的账户中扣款`​​​
          * 如果订单详情状态`OrderDetailStatusInfo`​​​为`待授权`​​​且为闪住`isFlashCredit`​​​，则展示配置文案（key=`FLASH_CREDIT_AUTHROZING_FEE_DESC`​​​），兜底：`授权时无需支付房费，离店后再从您授权的账户中扣款`​​​
          * 如果订单状态`orderData.orderBase.orderStatus`​​​为`CANCELLED`​​​:

            * 芝麻分：：

              * 金额 `orderData.fundOrder`​​​ + 汇率 `orderData.orderProduct.currencyExchange`​​​这两方面组合计算出来的用户实际支付的所有费用，包括但不限于：房费，组合支付费用，保险费用，快递费用
              * 如果金额大于0，rule=`取消订单将自动扣款¥%s`​​​（上述金额）
            * 否则：

              * 如果还款单`repayOrder`​​​不为null：还款金额来自于支付流水`repayOrder.payTraces.get("REPAY").amount`​​​
              * 否则来自于待履约金额`parent.tradeAuth.fulfillment.fulfillFees.amount加总`​​​
              * 如果金额大于0，rule = `取消订单将自动扣款¥%s`​​​（上述金额）
            * 如果上述金额为0，则展示`订单已取消，担保自动撤销`​​​
          * 如果是先住后付`isPayLater`​​​ 或者 闪住`isFlashCredit`​​​

            * 获取杂费金额：

              如果是后付授权（`MainOrder.OrderControlInfo.OrderAuthInfo.authPlatform.AuthType`​​​=`TRADE_AUTH`​​​）则取履约（`fulfillFees`​​​）否则取`orderData.miscFeeOrder`​​​
            * 如果杂费金额>0:

              * 如果是闪住：文案从配置中取，兜底：`扣款于离店日%s开始发起，将包含您在房间内的消费¥%s`​​​，金额=杂费金额
              * 否则：文案从配置中取，兜底：`扣款于离店日%s10:00后发起，将包含您在房间内的消费¥%s`​​​，金额=杂费金额
            * 如果杂费金额=0，无杂费：

              * 如果是闪住：文案从配置中取，兜底：`扣款于离店日%s开始发起`​​​
              * 非闪住：`扣款于离店日%s10:00后发起`​​​
          * 否则：`授权后可享受平台授信担保，房间整晚保留`​​​

          * 根据上述参数组装，
        * 否则：

          * 已授权 或 已支付：`担保金%s用于保留房间，离店后自动原路退`​​，金额 = 担保金额
          * 否则：​`担保后房间整晚保留，离店后担保金自动原路退`​​
      * 如果未授权且未支付 且 需要信用担保，增加：

        * ​`overviewFees`​​

          * text = text（上文中的）
          * price = ​`guaraPriceDesc`​​
          * desc = ​`unPayDesc`​​
          * ​`ruleTips`​​ = rule
          * show
      * 否则
    * 增加`overviewFee：`​

      * text = ​`到店付`​
      * 如果是信用授权则展示上述rule，否则不展示

### 现付非担保

* text = ​`到店付`​​
* price  = price
* desc = null；
* rule = null；
* popTips = 配置，key取​`cash`​​
* showButton = 订单状态==​`GUARANTEEING`​​

### 规则文案的其它处理

* 构造fan'x

## 支付方式下扣款文案寄状态的来源与逻辑

1. 所用字段progressInfo.tips
2. 退款进度文案及状态变化

    * 有开关控制
    * 信用住、先住后付、信用担保都没有退款进度信息，
    * 根据订单号查询交易，获取展示数据，将以下内容加入List：

      * 如果展示，依据`ShowResponse.display`​​:

        * 如果订单状态为`CANCELLED`​​，则展示`订单取消，支付金额将原路退回`​​，
        * 否则，展示`担保金将退回您的账户`​​
        * 加入List
      * 如果是后付类型：

        * 查看履约状态`parentOrder.tradeAuth.fulfillment.processStatus`​​

          * ​`processStatus`​​= `ACCEPT`​​扣款中：

            * title = `扣款中`​​
            * tip =

              * 如果授权渠道（`orderData.parentOrder.payChannel.authType如果为null则走老逻辑兼容`​​）不为空

                * 如果是信用担保订单：

                  * ​`授信担保扣款中，您未入住，将从您授权的【授权渠道】扣除担保金`​​
                * 否则：

                  * ​`优先从您授权的【授权渠道】扣款`​​
              * 否则

                * 如果是信用授权：

                  * ​`授信担保扣款中，账单已生成，将从您授权的账户扣除担保金`​​
                * 否则：

                  * ​`将从您授权的账户扣款`​​
          * ​`processStatus`​​=`SUCCESS`​​扣款成功：

            * title = `扣款成功`​​
            * tip = 

              * 如果是信用担保订单：`授信担保扣款成功 ，已从您的【银行卡/微信/支付宝/拿去花/授权账户】扣款`​​
              * 否则：`已从您的【银行卡/微信/支付宝/拿去花/授权账户】扣款`​​
          * ​`processStatus`​=​`PAY_FAIL`​扣款失败

            * title = 扣款失败
            * tips =

              * 如果是信用担保订单：

                * 展示：`"授信担保扣款失败，请尽快操作还款，否则将影响您的征信及后续使用`​再追加：

                  * 先住后付 或 程信分信用住订单：`先住后付服务`​  
                    先住后付判断依据：`mainOrder.orderControlInfo.orderCreditType = PAY_LATER/FLASH_CREDIT`​  
                    程信分判断依据：`mainOrder.orderControlInfo.orderAuthInfo.authPlatform=qunar`​
                  * 信用担保订单：​`免担保金服务`​  
                    判断依据：`mainOrder.orderControlInfo.orderCreditType`​
                  * 后付授权：`先住后付服务`​  
                    判断依据：`mainOrder.orderControlInfo.orderAuthInfo.authPlatform`​=​`TRADE_AUTH`​
                  * 兜底：

                    * ​`授权服务`​
              * 否则：

                * 展示：​`请尽快操作还款，否则将影响您的征信及后续使用`​
      * 如果是程信分信用住订单 或 信用担保订单 或 程信分先住后付订单

        * ​`orderData.repayOrder.payTraces取key = REPAY/debt.getStatus:`​

          * ​`Status`​=`PAY`​:

            * key = `REPAY`​，还款金额>0展示  
              title = ：`扣款中`​  
              tip=

              * 授权渠道不为空

                * 信用担保订单 = `授信担保扣款中，您未入住，将从您授权的" +【授权渠道】+ "扣除担保金`​
                * 否则 = `优先从您授权的" + 【授权渠道】 + "扣款`​
              * 授权渠道为空

                * 信用担保 = ​`授信担保扣款中，账单已生成，将从您授权的账户扣除担保金`​
                * 否则 = ​`将从您授权的账户扣款`​
            * 还款金额为0，则不展示
          * ​`Status`​=`PAY_SUCCESS`​

            * key = `DEBT`​，还款金额>0 且 repayOrder.status ！= ​`PAID全额支付成功`​

              * title=​`扣款失败`
              * tip=

                * 债务`debt`​ < 还款 `repay`​（部分垫付后，用户欠款情况下的进度说明文案）:`已扣款¥%.2f元，还需还款¥%.2f元，请尽快还款，逾期未还，将影响您后续使用`​+【`先住后付服务`​/`免担保金服务`​/`先住后付服务/授权服务`​】
                * 否则（扣款失败，垫付0元情况下的进度说明文案）：

                  * 信用担保订单=`授信担保扣款失败，请尽快操作还款，否则将影响您的征信及后续使用`​
                  * 否则 = ​`请尽快操作还款，否则将影响您的征信及后续使用`​
        * 否则 

          * 返回null
        * 加入List
      * 如果订单状态 = `CHECKED_OUT`​  
        有杂费欠款，没有房费欠款时展示

        * title = ​`扣款失败`​
        * tip = ​`房间内消费扣款失败`​
      * 如果可申请返现，判断依据订单状态!=`CANCELLED`​&`NO_SHOW`​ & `REJECTED`​且 返现金额>0。

        * title = ​`可返现`​
        * tip = `离店后30天内，可领返现%s%s`​返现金额 + `（含已兑换券）`​如果兑换过返现商品  
          兑换返现商品的依据：`orderData.mainOrder.orderExtension.secondStatus.cashRedeemApply.applyStatus`​
      * 如果是返现单：

        * ​`orderData.getMainPay().cashPay.getCashStatus`​ =`CASH_BACK_SUCCESS`​：返现成功

          * title = ​`返现成功`​
          * tip =​`可从您的去哪儿余额帐户查询`​
        * `orderData.getMainPay().cashPay.getCashStatus`​ =`CASH_BACK：`​待返现

          * title = ​`返现中`​
          * tip = `预计`​x月+y`日发放到去哪儿余额帐户（以实际返现为准）`​
        * ​`orderData.getMainPay().cashPay.getCashStatus`​ =`START：`​待返现

          * title = ​`返现中`​
          * tip = ​`预计`​x月+y`日发放到去哪儿余额帐户（以实际返现为准）`​
          * 上下两个不同的待返现，区别在于金额计算有所区别

‍
