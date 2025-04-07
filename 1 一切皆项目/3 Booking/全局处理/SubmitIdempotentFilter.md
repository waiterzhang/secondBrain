# SubmitIdempotentFilter

* 重复提交验证

  * 查看有无B参，无B参非重复提交；
  * 重复提交看起来使用redis做的。每个提交的请求都会有一个幂等的标识，这个标识的主要组成是`SUBMIT_IDEMPOTENT_`​，以及trace或者客户端幂等标识联合组成。
  * 如果查的到说明在一定时间内重复提交过。业务上的重复提交还需要验证上一次的提交状态时正常的，会判断status = 0，或者bstatus = 0。

‍

‍
