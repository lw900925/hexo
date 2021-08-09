---
title: Java 8新特性（二）：Stream API
date: 2016-12-29
desc:
keywords: java8
categories: [java]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/java8-logo.jpeg">

本篇介绍Java 8的另一个新特性——Stream API。新增的Stream API与`InputStream`和`OutputStream`是完全不同的概念，Stream API是对Java中集合操作的增强，可以利用它进行各种过滤、排序、分组、聚合等操作。

Stream API配合Lambda表达式可以加大的简化代码，提升可读性。Stream API也支持并行操作（类似于Fork-Join），甚至不用手动编写多线程代码，Stream API已经帮我们做好了，并且能充分利用多核CPU的优势。借助Stream API和Lambda表达式，可以很容易的编写出高性能的并发处理程序。

<!-- more -->

## Stream API简介

Stream API是Java 8中加入的一套新的API，主要用于处理集合操作，不过它的处理方式与传统的方式不同，称为“数据流处理”。流（Stream）类似于关系数据库的查询操作，是一种声明式操作。比如要从数据库中获取所有年龄大于20岁的用户的名称，并按照用户的创建时间进行排序，用一条SQL语句就可以搞定，不过使用Java程序实现就会显得有些繁琐，这时候可以使用流：

```java
List<String> userNames =
        users.stream()
        .filter(user -> user.getAge() > 20)
        .sorted(comparing(User::getCreationDate))
        .map(User::getUserName)
        .collect(toList());
```

可以把流跟集合做一个比较。在Java中，集合是一种数据结构，或者说是一种容器，用于存放数据，流不是容器，它不关心数据的存放，只关注如何处理。可以把流当做是Java中的`Iterator`，不过它可比`Iterator`强大多了。

流与集合另一个区别在于他们的遍历方式，遍历集合通常使用`for-each`方式，这种方式称为**外部迭代**，而流使用**内部迭代**方式，也就是说它帮你把迭代的工作做了，你只需要给出一个函数来告诉它接下来要干什么：

```java
// 外部迭代
List<String> list = Arrays.asList("A", "B", "C", "D");
for (String str : list) {
    System.out.println(str);
}

// 内部迭代
list.stream().forEach(System.out::println);
```

在一些比较复杂的业务场景中，要对集合做一些统计、分组的操作，如果用传统的`for-each`方式遍历集合，每次只能处理一个元素，并且是按顺序处理，这种方法是极其低效的。你可能会想到用多线程去并行处理，但是编写多线程代码并非易事，容易出错并且维护困难。不过在Java 8之后，你可以使用Stream API来处理这些操作。

Stream API将迭代操作封装到了内部，它会自动的选择最优的迭代方式，并且使用并行方式处理时，将集合分成多段，每一段分别使用不同的线程处理，最后将处理结果合并输出，有点类似于Fork-Join操作。

需要注意的是，流只能遍历一次，遍历结束后，这个流就被关闭掉了。如果要重新遍历，可以从数据源（集合）中重新获取一个流。如果你对一个流遍历两次，就会抛出`java.lang.IllegalStateException`异常：

```java
List<String> list = Arrays.asList("A", "B", "C", "D");
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
stream.forEach(System.out::println); // 这里会抛出java.lang.IllegalStateException异常，因为流已经被关闭
```

流通常由三部分构成：
1. 数据源：数据源一般用于流的获取，比如本文开头那个过滤用户的例子中`users.stream()`方法。
2. 中间处理：中间处理包括对流中元素的一系列处理，如：过滤（`filter()`），映射（`map()`），排序（`sorted()`）。
3. 终端处理：终端处理会生成结果，结果可以是任何不是流值，如`List<String>`；也可以不返回结果，如`stream.forEach(System.out::println)`就是将结果打印到控制台中，并没有返回。

## 创建流

创建流的方式有很多，具体可以划分为以下几种：

### 由值创建流

使用静态方法`Stream.of()`创建流，该方法接收一个变长参数：

```java
Stream<Stream> stream = Stream.of("A", "B", "C", "D");
```

也可以使用静态方法`Stream.empty()`创建一个空的流：

```java
Stream<Stream> stream = Stream.empty();
```

### 由数组创建流

使用静态方法`Arrays.stream()`从数组创建一个流，该方法接收一个数组参数：

```java
String[] strs = {"A", "B", "C", "D"};
Stream<Stream> stream = Arrays.stream(strs);
```

### 通过文件生成流

使用`java.nio.file.Files`类中的很多静态方法都可以获取流，比如`Files.lines()`方法，该方法接收一个`java.nio.file.Path`对象，返回一个由文件行构成的字符串流：

```java
Stream<String> stream = Files.lines(Paths.get("text.txt"), Charset.defaultCharset());
```

### 通过函数创建流

`java.util.stream.Stream`中有两个静态方法用于从函数生成流，他们分别是`Stream.generate()`和`Stream.iterate()`：

```java
// iteartor
Stream.iterate(0, n -> n + 2).limit(51).forEach(System.out::println);

// generate
Stream.generate(() -> "Hello Man!").limit(10).forEach(System.out::println);
```

第一个方法会打印100以内的所有偶数，第二个方法打印10个`Hello Man!`。需要注意的是，这两个方法生成的流都是无限流，没有固定大小，可以无穷的计算下去，在上面的代码中我们使用了`limit()`来避免打印无穷个值。

一般来说，`iterate()`用于生成一系列值，比如生成以当前时间开始之后的10天的日期：

```java
Stream.iterate(LocalDate.now(), date -> date.plusDays(1)).limit(10).forEach(System.out::println);
```

`generate()`方法用于生成一些随机数，比如生成10个UUID：

```java
Stream.generate(() -> UUID.randomUUID().toString()).limit(10).forEach(System.out::println);
```

## 使用流

`Stream`接口中包含许多对流操作的方法，这些方法分别为：

- `filter()`：对流的元素过滤
- `map()`：将流的元素映射成另一个类型
- `distinct()`：去除流中重复的元素
- `sorted()`：对流的元素排序
- `forEach()`：对流中的每个元素执行某个操作
- `peek()`：与`forEach()`方法效果类似，不同的是，该方法会返回一个新的流，而`forEach()`无返回
- `limit()`：截取流中前面几个元素
- `skip()`：跳过流中前面几个元素
- `toArray()`：将流转换为数组
- `reduce()`：对流中的元素归约操作，将每个元素合起来形成一个新的值
- `collect()`：对流的汇总操作，比如输出成`List`集合
- `anyMatch()`：匹配流中的元素，类似的操作还有`allMatch()`和`noneMatch()`方法
- `findFirst()`：查找第一个元素，类似的还有`findAny()`方法
- `max()`：求最大值
- `min()`：求最小值
- `count()`：求总数

下面逐一介绍这些方法的用法。

### 过滤和排序

```java
Stream.of(1, 8, 5, 2, 1, 0, 9, 2, 0, 4, 8)
    .filter(n -> n > 2)     // 对元素过滤，保留大于2的元素
    .distinct()             // 去重，类似于SQL语句中的DISTINCT
    .skip(1)                // 跳过前面1个元素
    .limit(2)               // 返回开头2个元素，类似于SQL语句中的SELECT TOP
    .sorted()               // 对结果排序
    .forEach(System.out::println);
```

### 查找和匹配

Stream中提供的查找方法有`anyMatch()`、`allMatch()`、`noneMatch()`、`findFirst()`、`findAny()`，这些方法被用来查找或匹配某些元素是否符合给定的条件：

```java
// 检查流中的任意元素是否包含字符串"Java"
boolean hasMatch = Stream.of("Java", "C#", "PHP", "C++", "Python")
        .anyMatch(s -> s.equals("Java"));

// 检查流中的所有元素是否都包含字符串"#"
boolean hasAllMatch = Stream.of("Java", "C#", "PHP", "C++", "Python")
        .allMatch(s -> s.contains("#"));

// 检查流中的任意元素是否没有以"C"开头的字符串
boolean hasNoneMatch = Stream.of("Java", "C#", "PHP", "C++", "Python")
        .noneMatch(s -> s.startsWith("C"));

// 查找元素
Optional<String> element = Stream.of("Java", "C#", "PHP", "C++", "Python")
        .filter(s -> s.contains("C"))
        // .findFirst()     // 查找第一个元素
        .findAny();         // 查找任意元素
```

注意最后一行代码的返回类型，是一个`Optional<T>`类（`java.util.Optional`），它一个容器类，代表一个值存在或不存在。上面的代码中，`findAny()`可能什么元素都没找到。`Optional<T>`是Java 8的另一个新特性，有关`Optional<T>`类的详细用法，将在下一篇文章中介绍。

实际上测试结果发现，`findFirst()`和`findAny()`返回的都是第一个元素，那么两者之间到底有什么区别？通过查看javadoc描述，大致意思是`findAny()`是为了提高并行操作时的性能，如果没有特别需要，还是建议使用`findAny()`方法。

### 归约

归约操作就是将流中的元素进行合并，形成一个新的值，常见的归约操作包括求和，求最大值或最小值。归约操作一般使用`reduce()`方法，与`map()`方法搭配使用，可以处理一些很复杂的归约操作。

```java
// 获取流
List<Book> books = Arrays.asList(
       new Book("Java编程思想", "Bruce Eckel", "机械工业出版社", 108.00D),
       new Book("Java 8实战", "Mario Fusco", "人民邮电出版社", 79.00D),
       new Book("MongoDB权威指南（第2版）", "Kristina Chodorow", "人民邮电出版社", 69.00D)
);

// 计算所有图书的总价
Optional<Double> totalPrice = books.stream()
       .map(Book::getPrice)
       .reduce((n, m) -> n + m);

// 价格最高的图书
Optional<Book> expensive = books.stream().max(Comparator.comparing(Book::getPrice));
// 价格最低的图书
Optional<Book> cheapest = books.stream().min(Comparator.comparing(Book::getPrice));
// 计算总数
long count = books.stream().count()
```

在计算图书总价的时候首先使用`map()`方法得到所有图书价格的流，然后再使用`reduce()`方法进行归约计算。与`map()`方法类似的还有一个`flatMap()`，通过名字可以看出`flatMap()`方法是将流进行扁平化操作，看看下面的代码：

```java
List<User> users = userDAO.selectAllByInstId(instId);
List<List<Role>> userRoles = users.stream()
        .map(User::getOid)
        .map(userId -> {
            List<Role> roles = roleDAO.selectAllByUserId(userId);
            return roles;
        }).collect(Collectors.toList());
```

首先根据`InstId`查询所有用户集合，再根据用户获取与其关联的所有角色集合，最终返回结果是一个`List<List<Role>>`类型，但这不是我们想要的类型，我们想要的是`List<Role>`这种扁平的集合，这时候`flatMap()`就派上用场了：

```java
List<User> users = userDAO.selectAllByInstId(instId);
List<Role> userRoles = users.stream()
        .map(User::getOid)
        .map(userId -> {
            List<Role> roles = roleDAO.selectAllByUserId(userId);
            return roles;
        }).flatMap(roles -> roles.stream()) // 扁平化操作
        .collect(Collectors.toList());
```

第二个`map()`返回的结果是`List<Role>`，接着使用`flatMap(roles -> roles.stream())`扁平化处理，将各个不同的`List<Role>`合并成一个大的`List<Role>`。`flatMap()`适合处理类似于二维数组这种格式，将其扁平化：

```java
List<List<String>> list = new ArrayList<List<String>>() {{
    add(new ArrayList<String>() {{
        add("A1");
        add("A2");
        add("A3");
    }});

    add(new ArrayList<String>() {{
        add("B1");
        add("B2");
        add("B3");
    }});

    add(new ArrayList<String>() {{
        add("C1");
        add("C2");
        add("C3");
    }});
}};

List<String> result = list.stream()
        .flatMap(ls -> ls.stream())
        .collect(Collectors.toList());
```

## 数据收集

前面两部分内容分别为流式数据处理的前两个步骤：从数据源创建流、使用流进行中间处理。下面我们介绍流式数据处理的最后一个步骤——数据收集。

数据收集是流式数据处理的终端处理，与中间处理不同的是，终端处理会消耗流，也就是说，终端处理之后，这个流就会被关闭，如果再进行中间处理，就会抛出异常。数据收集主要使用`collect`方法，与`reduce()`方法一样，该方法也属于归约操作，将流中的元素累积成一个汇总结果，具体的做法是通过定义新的`Collector`接口来定义的。

在前面部分的例子中使用收集器（`Collector`）是由`java.util.stream.Collectors`工具类中的`toList()`方法提供，`Collectors`类提供了许多常用的方法用于处理数据收集，常见的有归约、汇总、分组等。

### 归约和汇总

我们使用前面归约操作中计算图书总价，最大值，最小值，输入总数那个例子来看看收集器如何进行上述归约操作：

```java
// 求和
long count = books.stream().collect(counting());

// 价格最高的图书
Optional<Book> expensive = books.stream().collect(maxBy(comparing(Book::getPrice)));

// 价格最低的图书
Optional<Book> cheapest = books.stream().collect(minBy(comparing(Book::getPrice)));
```

上面的代码需要使用静态导入`Collectors`和`Comparator`两个类，这样你就不用再去写`Collectors.counting()`和`Comparator.comparing()`这样的代码了：

```java
import static java.util.stream.Collectors.*;
import static java.util.Comparator.*;
```

`Collectors`工具类为我们提供了用于汇总的方法，包括`summarizingInt()`，`summarizingLong()`和`summarizingDouble()`，由于图书的价格为`Double`类型，所以我们使用`summarizingDouble()`方法进行汇总。该方法会返回一个`DoubleSummaryStatistics`对象，包含一系列归约操作的方法，如：汇总、计算平均数、最大值、最小值、计算总数：

```java
DoubleSummaryStatistics dss = books.stream().collect(summarizingDouble(Book::getPrice));
double sum = dss.getSum();          // 汇总
double average = dss.getAverage();  // 求平均数
long count = dss.getCount();        // 计算总数
double max = dss.getMax();          // 最大值
double min = dss.getMin();          // 最小值
```

`Collectors`类还包含一个`joining()`方法，该方法用于连接字符串：

```java
String str = Stream.of("A", "B", "C", "D").collect(joining(","));
```

上面的代码用于将流中的字符串通过逗号连接成一个新的字符串。

### 分组

和关系数据库一样，流也提供了类似于数据库中`GROUP BY`分组的特性，由`Collectors.groupingBy()`方法提供：

```java
Map<String, List<Book>> booksGroup = books.stream().collect(groupingBy(Book::getPublisher));
```

上面的代码按照出版社对图书进行分组，分组的结果是一个`Map`对象，`Map`的`key`值是出版社的名称，`value`值是每个出版社分组对应的集合。分组方法`groupingBy()`接收一个`Function`接口作为参数，上面的例子中我们使用了方法引用传递了出版社作为分组的依据，但实际情况可能比这复杂，比如将价格在0-50之间的书籍分成一组，50-100之间的分成一组，超过100的分成一组，这时候，我们可以直接使用Lambda表达式来表示这个分组逻辑：

```java
Map<String, List<Book>> booksGroup = books
    .stream()
    .collect(groupingBy(book -> {
        if (book.getPrice() > 0 && book.getPrice() <= 50) {
            return "A";
        } else if (book.getPrice() > 50 && book.getPrice() <=100) {
            return "B";
        } else {
            return "C";
        }
    }));
```

`groupingBy()`方法还支持多级分组，他有一个重载方法，除了接收一个`Function`类型的参数外，还接收一个`Collector`类型的参数：

```java
Map<String, Map<String, List<Book>>> booksGroup = books.stream().collect(
        groupingBy(Book::getPublisher, groupingBy(book -> {
            if (book.getPrice() > 0 && book.getPrice() <= 50) {
                return "A";
            } else if (book.getPrice() > 50 && book.getPrice() <=100) {
                return "B";
            } else {
                return "C";
            }
        }))
);
```

上面的代码将之前两个分组合并成一个，实现了多级分组，首先按照出版社进行分组，然后按照价格进行分组，返回类型是一个`Map<String, Map<String, List<Book>>>`。`groupingBy()`的第二个参数可以是任意类型，只要是`Collector`接口的实例就可以，比如先分组，再统计数量：

```java
Map<String, Long> countGroup = books.stream()
        .collect(groupingBy(Book::getPublisher, counting()));
```

还可以在进行分组后获取每组中价格最高的图书：

```java
Map<String, Book> expensiveGroup = books.stream()
        .collect(groupingBy(Book::getPublisher, collectingAndThen(
            maxBy(comparingDouble(Book::getPrice)),
                Optional::get
        )));
```

## 并行数据处理

### 并行流

并行流使用集合的`parallelStream()`方法可以获取一个并行流。Java内部会将流的内容分割成若干个子部分，然后将它们交给多个线程并行处理，这样就将工作的负担交给多核CPU的其他内核处理。

我们通过一个简单粗暴的例子演示并行流的处理性能。假设有一个方法，接受一个数字n作为参数，返回从1到n的所有自然数之和：

```java
public static long sequentialSum(long n) {
	return Stream.iterate(1L, i -> i + 1)
			.limit(n)
			.reduce(0L, Long::sum);
}
```

上面的方法也可以通过传统的for循环方式实现：

```java
public static long iterativeSum(long n) {
	long result = 0;
	for (long i = 1L; i <= n; i++) {
		result += i;
	}
	return result;
}
```

编写测试代码：

```java
public static void main(String[] args) {
	long number = 10000000L;
    System.out.println("Sequential Sum: " + sumPerformanceTest(StreamTest::sequentialSum, number) + " 毫秒");
    System.out.println("Iterative Sum: " + sumPerformanceTest(StreamTest::iterativeSum, number) + " 毫秒");
}

public static long sumPerformanceTest(Function<Long, Long> function, long number) {
	long maxValue = Long.MAX_VALUE;

	for (int i=0; i<10; i++) {
		long start = System.nanoTime();
		long sum = function.apply(n);
		long end = System.nanoTime();
		System.out.println("Result: " + sum);
		long time = ( end - start ) / 1000000;

		if (time < maxValue) {
			maxValue = time;
		}
	}

	return maxValue;
}
```

上面的测试代码中我们编写了一个`sumPerformanceTest()`方法，参数`number`表示给定的一个数，用于计算从1到这个数的所有自然数之和。该方法内部执行10次运算，返回时间最短的一次运算结果。

运行上面的代码，可以在控制台看到如下结果：

```
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Sequential Sum: 159 毫秒
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Iterative Sum: 5 毫秒
```

从结果来看，采用传统的for循环更快，因为它不用做任何自动拆箱/装箱操作，操作的都是基本类型。这个测试结果并不客观，提升的性能取决于机器的配置，以上是我在公司的台式机（机器配置为`Intel(R) Core i7-6700 CPU 3.40HZ; 8GB RAM`）上运行的结果。

现在我们使用并行流测试一下：

```java
public static long parallelSum(long n) {
	return Stream.iterate(1L, i -> i + 1)
			.limit(n)
			.parallel()
			.reduce(0L, Long::sum);
}

public static void main(String[] args) {
	System.out.println("Parallel Sum: " + sumPerformanceTest(StreamTest::parallelSum, number) + " 毫秒");
}
```

并行流执行结果为：

```
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Parallel Sum: 570 毫秒
```

并行的执行效率比顺序执行还要慢，这个结果有点出乎意料。主要有两个原因：

1. `iterate()`方法生成的对象是基本类型的包装类（也就是`java.lang.Long`类型），必须进行拆箱操作才能运算。
2. `iterate()`方法不适合用并行流处理。

第一个原因容易理解，自动拆箱操作确实需要花费一定的时间，这从前一个例子可以看出来。

第二个原因中`iterate()`方法不适合用并行流处理，主要原因是`iterate()`方法内部机制的问题。`iterate()`方法每次执行都需要依赖前一次的结果，比如本次执行的输入值为10，这个输入值必须是前一次运算结果的输出，因此`iterate()`方法很难使用并行流分割成不同小块处理。实际上，上面的并行流程序还增加了顺序处理的额外开销，因为需要把每次操作执行的结果分别分配到不同的线程中。

一个有效的处理方式是使用`LongStream.rangeClosed()`方法，该方法弥补了上述例子的两个缺点，它生成的是基本类型而非包装类，不用拆箱操作就可以运算，并且，它生成的是有范围的数字，很容易拆分。如：生成1-20范围的数字可以拆分成1-10, 11-20。

```java
public static long rangedSum(long n) {
	return LongStream.rangeClosed(1, n)
			.reduce(0L, Long::sum);
}

public static void main(String[] args) {
	System.out.println("Ranged Sum: " + sumPerformanceTest(StreamTest::rangedSum, number) + " 毫秒");
}
```

执行结果为：

```
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Ranged Sum: 8 毫秒
```

这个结果比起`sequentialSum()`方法执行的结果还要快，所以选择合适的数据结构有时候比并行化处理更重要。我们再将`rangeClosed()`方法生成的流转化为并行流：

```java
public static long parallelRangedSum(long n) {
    return LongStream.rangeClosed(1, n)
            .parallel()
            .reduce(0L, Long::sum);
}

public static void main(String[] args) {
	System.out.println("Parallel Ranged Sum: " + sumPerformanceTest(StreamTest::parallelRangedSum, number) + " 毫秒");
}
```

执行结果为：

```
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Result: 200000010000000
Parallel Ranged Sum: 2 毫秒
```

我们终于得到了想要的结果，所以并行操作需要选择合适的数据结构，建议多做测试，找到合适的并行方式再执行，否则很容易跳到坑里。
