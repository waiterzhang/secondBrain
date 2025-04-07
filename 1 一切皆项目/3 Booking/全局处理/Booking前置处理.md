# Booking前置处理

## web.xml

配置了一些filter和

* ​`CharacterEncodingFilter`​  
  显然是一个字符编码工作。
* ​`ServletWatcher`​​

  * 实现了ServletContextListener和Filter  
    ServletContextListener针对的是Servlet容器的启动和关闭，Filter针对的是请求。两者不一致。
  * 这个似乎是做了一些服务器连接等非常前置的内容。
* ​`RequestContextFilter`​​​​  
  RequestContextFilter的作用是为每个请求创建一个请求上下文，并将其绑定到当前线程上，以便在整个请求处理过程中共享数据。它是实现请求级别的数据共享和管理的重要组件。
* ​`DelegatingFilterProxy`​：

  对于Serlvet filter的代理，用这个类的好处就是可以通过Spring容器来管理servlet filter的生命周期。如果filter中需要一些spring容器的示例，可以通过spring直接注入。

  这个将Filter与spring的ioc结合起来。
* ​`HttpServletAccessQtraceFilter`​：

  ​`处理异步 HTTP Servlet 请求的跨线程执行时的 QTrace 以及特定 MDC 值的延续问题。`​
* ​`UsingAppResultFilter`​​​

  在resp的header中打标记，key=`using.app.result";`​value = ​`hbooking`​
* ​``DispatcherServlet``​  
  核心：  
  ​![image](image-20240116141446-rz2p4a0.png)​
* ​`ClientRawAccess`​  
  总结：设置C参、处理转发

  * qrt到url的映射；
  * header的标记，key = `client.raw.access`​，value = ​`hbooking`​
  * 设置c参、v参？、uid空且gid不空，则将gid赋值给uid（gid是设备号）、设置ip地址（首选request里面ip、次选c参里面的userip）、替换cqp（`RN版本号`​）、打时间戳等。  
    主要是设置、替换c参
* ​`ResponseDiffAccess`​  
  处理历史协议之间切换所导致的问题，现已废弃。
