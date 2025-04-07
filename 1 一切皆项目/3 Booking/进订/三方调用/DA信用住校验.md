# DA信用住校验

* 首先从报价的extMap中取信用类型：  
  ​![image](image-20240507142929-hjyaetn.png)​
* 把一些预定信息（入离日期、酒店信息、用户信息）给到风控

用处：

* 先主后付会校验信用住校验是否通过，如果不通过则走预防。
* 先住后付有开关，开关的key使用端+先住后付类型拼接成
* 风控校验结果会放入闪住信息中`flashLoading.setCreditCheckType(creditCheckResult.name())`​
* 关于一部分先住后付的逻辑，还得再看看

‍
