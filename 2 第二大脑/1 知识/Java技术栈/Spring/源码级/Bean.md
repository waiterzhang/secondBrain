### QA
从一个Context里面getBean的时候的时候发生了什么？这些Bean是如何到Context里面去的？

Q1、初始化一个Context的容器。需要知道哪些类我需要初始化，我咋知道呢？
- 需要传入一个Config的配置类，在这个配置类里面指定哪些包下面的类我们需要去扫描。这个具体的配置通过@ComponentScan注解的value的方式来配置。
- 拿到具体的包（com.zwt.service）以后还不够，我们需要通过这个包名转换到一个具体的文件夹格式（com/zwt/service），如此还只是相对路径，同时我们需要拿到的是class文件的路径。最终由classLoader相关方法拿到编译文件的路径。（此处不是重点）  
    总之，通过一系列的方法，我可以拿到指定类的全限定类名。
- 当拿到全限定类名后，我就可以拿到对应的Class对象。通过Class对象，我可以知道哪些类上面有@Component注解，标注了此注解的对象肯定是要注入的对象。

Q2、知道了哪些类要生成，立刻就要生成吗？什么时候生成？
- 非也。生成一个类有许多要考虑的地方，最简单的就是多例or单例？可是如何拿到一个类是单例还是多例的信息呢？这个信息可以再通过一个注解去指定它的@Scope。
- 知道了上述信息也不要直接生成，直接生成太死板了，很难更细化的控制。所以所有类的信息可以定义为一个BeanDefinition，把这个信息存储起来。
- 有了单例和多例信息，单例对象自然可以进行缓存or池化。多例对象获取的时候再去new即可。单例可以在容器初始化的时候就生成，单例实现的细节不是重点，略。

Q3、有了bean，如何去get呢？
- 单例先从单例池里面取，没有就new，new出来的记住放到单例池子里面。


### 生命周期
    
    - 推断构造方法
        
    - 构造后获得一个普通对象
        
    - 依赖注入
        - 同样先byType再byName
    - 初始化前
        
    - 初始化
        
    - 初始化后（AOP）
        
    - 代理对象
        
    - 放入Map（单例池）
        
    - Bean对象
        

- 如果一个类里面有多个构造方法，如果没有指定：如果有无参构造，优先无参，只有一个构造则使用该方法。这两条规则都是很明确的用哪个构造方法。如果没有无参构造，且有多个方法，就报错，因为spring也不知道该用哪个。  
    如果有多个构造方法，你想让spring用哪个构造方法，就在上面加一个@Autowired即可。但是如果你在两个构造方法上面都@Autowired，则又会报错。因为spring又不知道用哪个了。
    
- 构造方法使用的入参spring从哪里找？单例池。如果没有，就去创建。前提入参是一个bean。如果是个多例，则直接创建。
    
    > 单例单的是名字不是类型，名称唯一。
    
- 在BeanDefinitionMap中以什么为key去找具体的类呢？类型？名字？  
    先byType再byName。如果byName找不到，或者找到多个，则会报错。
    
- AOP后生成代理对象，代理对象没有其他属性，因为代理对象生成之后没有依赖注入。所以那些属性也不可能被赋值。
    
- cglib的原理：
    
    ```javascript
    class UserService {
    	private OrderService orderService;
    
    	public void test(){
    		sout(orderService)
    	}
    }
    
    class UserServiceProxy extends UserService {
    	public void test(){
    		// 切面逻辑@Before
    
    		// super.test();
    
    		// 切面逻辑@After
    	}
    }
    ```
    

这里并不是直接如上处理，因为super执行的时候其并没有orderService对象。那是怎么做的呢？

```javascript
class UserService {
	private OrderService orderService;

	public void test(){
		sout(orderService)
	}
}

class UserServiceProxy extends UserService {
	UserService target;
	public void test(){
		// 切面逻辑@Before

		// target.test();

		// 切面逻辑@After
	}
}
```

产生代理对象之后，会有一个原来的target对象。也就是代理对象执行的时候，执行原方法还是被代理对象（普通对象）的方法。普通对象是经过依赖注入的。
