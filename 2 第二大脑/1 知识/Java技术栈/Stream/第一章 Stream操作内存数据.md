# 第一章 Stream操作内存数据

## stream能干什么？

stream为jdk中众所周知的map-filter-reduce算法提供了实现。可以帮助你去高效地处理内存数据。如果掌握了stream，你可以写出来更可读、更具表现力的代码。

> stream有很多人不建议使用的一个重要理由就是“可读性差”；
>
> 现在官方说他更可读，我觉得还是挺有意思的。仔细想了想，两者说的都有道理。可读性差，“差”这是一个比较词，自然需要先定义比较的对象。说stream可读性差，是将stream与显式的过程性的编程相比，那么新的api、新的编码风格确实变化非常大。基于这种想法那自然可读性变差了。
>
> 而官方文档中说可读性好，似乎指这种stream的编程风格更接近于自然语言。不在意过程，而更看重结果。

## 场景引入

假设有一个销售相关的任务需要做，有类如下：

```java
public class Sale {
    private String product;
    private LocalDate date;
    private int amount;

    // constructors, getters, setters
    // equals, hashCode, toString
}
```

统计三月份的销售额：

```java
List<Sale> sales = ...; // this is the list of all the sales
int amountSoldInMarch = 0;
for (Sale sale: sales) {
    if (sale.getDate().getMonth() == Month.MARCH) {
        amountSoldInMarch += sale.getAmount();
    }
}
System.out.println("Amount sold in March: " + amountSoldInMarch);
```

统计的可以分解为三个步骤：

* 过滤出来三月份的数据对象；（filtering）
* 抽取出数据对象中的sale属性；（mapping）
* 将所有sale数据加总。（reducing）

第三步很像一个sql语言：

```java
select sum(amount)
from Sales
where extract(month from date) = 3;
```

比较java代码何sql，可以看出来风格是有区别的：

* sql只是使用了一个函数，告诉数据库“我需要求和”，至于具体如何求和，sql是不管的，这交由数据库来负责；

* java中的总和是通过累加算出来的，需要你一步一步的描述如何计算总和，最重要的是你得写的非常准确；

## 了解mapping-filtering-reducing

### mapping

mapping指的是映射、转换，mapping是一对一的转换：如果一个list中有10个对象，对该list执行完mapping之后，获得的list里面仍有10个对象。

对于Steam API，mapping本身会有更多的限制，假设你想去处理一个有序的集合，mapping操作的顺序与有序集合一致。

> mapping可能改变对象的类型，但并不会改变对象的顺序

mapping由Function函数式接口支持。Function接口可以接收任意类型的参数，并返回任意类型的返回值。

### filtering

filtering本身并不会去接触对象本身，它只是去判断哪些对象应该去，哪些对象应该留。

> filtering只会改变对象的数量，并不会改变对象的类型

filtering由Pridicate函数式接口支持。Pridicate可以接收任意类型的入参，并返回bool类型。

### reducing

redcing（合并？归一？）比看起来更加棘手。现在， 我们将接受一个定义：reducing与sql中aggregation（聚合）是一类操作。类似的Count、Sum、Min、Max、Average的操作Stream API都支持。

reducing这一步允许你构建负责的结构，包括list、set、map或者其他类，只要你能构建出来的类型都可以。详细的细节，后面再讲。

## 优化mapping-filtering-reducing

假设有一个城市类，里面包含name何population两个字段，现在要计算超过100k人口的城市人口总和：

```java
List<City> cities = ...;

int sum = 0;
for (City city: cities) {
    int population = city.getPopulation();
    if (population > 100_000) {
        sum += population;
    }
}

System.out.println("Sum = " + sum);
```

为啥不直接再collection接口里面新增一些mapping-fitering-reduing方法呢？例如这样：

```java
int sum = cities.map(city -> city.getPopulation())
                .filter(population -> population > 100_000)
                .sum();
```

可读性看来很不错，那为啥collection接口里面没有map、fitler接口呢？

思考的更深一些，map、filter应该返回什么内容呢 ？考虑到是一个集合框架，这些函数返回一个集合似乎顺理成章，那么这两个函数可以写成这样：

```java
Collection<Integer> populations         = cities.map(city -> city.getPopulation());
Collection<Integer> filteredPopulations = populations.filter(population -> population > 100_000);
int sum                                 = filteredPopulations.sum();
```

代码看起来是正确的，分析一下这段代码：

* 第一步是mapping，如果要处理的数据有1000个城市，这一步将产生1000个整数值，并被存储值一个集合中。
* 第二步是filtering，这一步是过滤，将会产生一个新的集合，通过过滤的数据将会被放到这个新的集合里面。

很显然，这两步里面每一步都要产生一个新的集合，这个新的集合里面的元素可能是各种各样的类型、数量。如果处理的数据更多，步骤更加繁琐，中间过程会产生大量的新集合，**开销非常大。**

## Stream API

正确的方式应该是使用stream API：

```java
Stream<City> streamOfCities         = cities.stream();
Stream<Integer> populations         = streamOfCities.map(city -> city.getPopulation());
Stream<Integer> filteredPopulations = populations.filter(population -> population > 100_000);
int sum = filteredPopulations.sum(); // in fact this code does not compile; we'll fix it later
```

它有以下特点：

* stream的map和filter不会产生中间数据结构去存储中间对象，这使得stream非常高效。因此`treamOfCities`​, `populations`​ and `filteredPopulations`​ 都是空对象。**stream本身并不会存储数据（也有少数例外）。**

* stream被设计为：**只要不在Stream中创建非Stream对象，就不会进行数据的计算**。上面代码中只有sum = filteredPopulations.sum()的时候才触发了流的计算。

* 某些情况下，无需遍历集合中的所有元素，即可生成结果。

* stream如同管道一样被调用，返回stream的操作被称为中间操作，返回其他结果，包括void的操作被称为终止操作。中间操作一次处理一个元素，不可以一次调用多个中间、终结操作，这样会报错。

## 处理数字的Stream

如果需要处理数字，可以使用IntStream、LongStream、DoubleStream，这三个Stream里面使用的基础类型，而非包装类型的数字，其他方面与Stream几乎一样。由于处理的是数字，他们还有一些Stream中不存在的api：

* sum；
* min，max；
* average；
* sumaryStatics：这个方法会生成一个特殊的对象，其中包括很多统计信息，例如流处理的元素的数量、最大值、最小值、总和、平均值。

## 良好的使用习惯

* 不要将Stream存储在字段或者局部变量中，这本身无用且危险；
* 将Stream作为参数的方法可能也很危险，你无法确定这个Stream有没有被操作过，Stream应该当场创建、当场使用；
* Stream是连接源与目标的对象，它从这个源中提取对象，源本身不应该被Stream修改，这样会导致不可控的结果。因此在某些情况下，源应应该是不可变的、只读的。

‍
