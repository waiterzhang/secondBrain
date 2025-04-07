# 第二章 为Stream增加中间操作

## 将流转换到另一个流

使用map（），可以将流转换到另一个流。

map（）接收一个Function作为入参，映射到另外一个流意味着所有的所有的元素都会被该Function处理。

类似于这样：

```java
List<String> strings = List.of("one", "two", "three", "four");
Function<String, Integer> toLength = String::length;
Stream<Integer> ints = strings.stream()
                              .map(toLength);
```

map（String::length）创建了一个流Stream<Integer>，除此之外使用IntStream、LongStream、DoubleStream来指定

## 流的过滤

**引入一个场景**​

假设有一个state类，以及一个city类，彼此有包含关系。

```java
public class City {
  
    private String name;
    private int population;

    // constructors, getters
    // toString, equals and hashCode
}

public class State {
  
    private String name;
    private List<City> cities;

    // constructors, getters
    // toString, equals and hashCode
}
```

假设需要统计List<State>中包含的city的数量，可以这样写：

```java
List<State> states = ...;

int totalPopulation = 0;
for (State state: states) {
    for (City city: state.getCities()) {
        totalPopulation += city.getPopulation();
    }
}

System.out.println("Total population = " + totalPopulation);
```

上面代码中内循环加总的方式和之前的map-reduct操作类似，因此可以将其改写成流的形式：

```java
totalPopulation += state.getCities().stream().mapToInt(city::getPopulation).sum();
```

流放到循环里面不是很好，可以考虑看看flatmap。

> 我想了一下不好的原因：
>
> 流本身就是为了处理批量数据而生的，因此流中本身就有极强的loop性质，与loop循环写法并列存在。因此在loop里面套stream，有点左手筷子，右手刀叉的感觉。别扭。
>
> 感觉说的还是不够准确。

## Flatmap

flatmap可以在上述环境中大展身手，它在对象与流之间开启了一个一对多的关系。

> 如何理解“一对多”？
>
> 首先看一下问题：用map/reducing连接流与loop，其中的1是外层循环的对象，多是内循环的对象，相当于直接在外层循环调用内循环的对象。将两层loop，变为一层。

flatmap接收一个Function作为参数。这个Function接受一个对象（相当于外层循环的对象），并且返回一个流（相当于该对象中集合的Stream）。

```java
Function<State, Stream<City>> stateToCity = state -> state.getCities().stream();
```

flatmap方法处理stream分为两步：

* 使用该函数映射流中的所有元素，从Stream<State>到Stream<Stream<City>>;
* 将Stream包裹的Straem“展平”。最终得到的将是**一个**包括所有city的流，而不是每个state拥有一个流，该流中又包含城市。

简而言之，flatmap本身和map本质是一样的，但又有其特点：

* faltmap首先是个map，map就得有function，这个function有其特殊性，必须得返回流。
* flatmap还有flat，falt是展平的意思，即将多个流的内容转换到一个流里面去。

如图所示：

​​​![image](image-20231118184450-4nds5wz.png)​

## 删除重复项并排序

Stream API有两个方法

* distinct：检测重复项，通过hashcode和equals；
* sorted：排序，需要传入一个比较器；

流不存储任何对象，但是上面两个是例外。为了发现重复项，distinct需要存储流的元素，当它处理元素的事后首先看一下是不是已经出现了，

sorted也需要在将其转化到下一个管道时，在内部缓冲区进行排序。

## 限制和跳过流的元素

Stream提供了两种选择元素的方式，基于索引，或者谓词（Predication）

基于索引，就是skip（跳过的索引）和limit（保留前n个元素）方法，两者入参都是long。两者使用上有一些陷阱需要避开。由于每次调用都是创建一个新的流，所以需要注意这一点。假设你想取整数流上3-8之间的数字，可以这样写：

```java
List<Integer> ints = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9);
List<Integer> result = 
    ints.stream()
        .skip(2)
        .limit(5)
        .collect(Collectors.toList());

System.out.println("result = " + result);
```

而不要skip(2).limit(8)

## 合并流

可以使用Stream.concat()来连接两个流，对于多个流，可以使用flatmap来实现多个流的连接：

```java
List<Integer> list0 = List.of(1, 2, 3);
List<Integer> list1 = List.of(4, 5, 6);
List<Integer> list2 = List.of(7, 8, 9);

// 1st pattern: concat
List<Integer> concat = 
    Stream.concat(list0.stream(), list1.stream())
          .collect(Collectors.toList());

// 2nd pattern: flatMap
List<Integer> flatMap =
    Stream.of(list0.stream(), list1.stream(), list2.stream())
          .flatMap(Function.identity())
          .collect(Collectors.toList());

System.out.println("concat  = " + concat);
System.out.println("flatMap = " + flatMap);
```

这两种方法又细微的区别：

concat对于数量是源与目标流的数量都是已知的，而flatmap则需要进行一些运算，对于stream中的数量是未知的。

## 调试流

peek方法可以比较方便的调试流，本质上是将一个Consumer接口当作参数。可以这样做调试，但是正式代码中不要这样写。

```java
List<String> strings = List.of("one", "two", "three", "four");
List<String> result =
        strings.stream()
                .peek(s -> System.out.println("Starting with = " + s))
                .filter(s -> s.startsWith("t"))
                .peek(s -> System.out.println("Filtered = " + s))
                .map(String::toUpperCase)
                .peek(s -> System.out.println("Mapped = " + s))
                .collect(Collectors.toList());
System.out.println("result = " + result);
```

‍

‍
