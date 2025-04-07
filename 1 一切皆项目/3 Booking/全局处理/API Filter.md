# API Filter

* 处理预售，

  * 从Request中取Pid、Vid、Uid等内容填充到request的userModel中
  * 通过request向用户中心查询用户信息。查不到，流程也不会阻断
  * 每看明白，MDC是干嘛的？似乎是控制日志打印，打印了一些用户设备信息；

‍
