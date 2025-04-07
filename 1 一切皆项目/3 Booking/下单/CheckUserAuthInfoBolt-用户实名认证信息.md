# CheckUserAuthInfoBolt-用户实名认证信息

* 有开关，默认为false
* 是否需要实名认证信息：（主要是为了防薅羊毛），以下三者，中一个即为需要实名校验。不需要实名校验，则不验证。

  * 春节券
  * 政府券：政府券可以配置需要身份证、真实姓名校验，两者命中其一，认为需要实名认证信息。
  * 是否有微信支付优惠券。
* mock身份判定，此处的身份可以通过配置进行mock

  > 这个可以方便开发
  >

* 查询携程，用户实名认证结果，url=`http://qunargateway.ctripgroup.cn/finance/apimerchantservice/queryUserAuthInfo"`​

  返回结果的结构：  
  ​![image](image-20240312192217-ze7l134.png)​

  针对返回结果监控，重点监控了idType、以及status了，status=0看起来是可用，stats = 100看起来是不可用。监控地址：  
  https://watcher.corp.qunar.com/dashboard/team/?path=qunar.team.hotel.a_frontend.user.hbooking.api.biz&from=now-1h&to=now  
  ​![image](image-20240312192428-r2upu4p.png)​
* ‍
