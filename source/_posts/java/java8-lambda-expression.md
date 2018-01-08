---
title: Java 8新特性（一）：Lambda表达式
date: 2016-12-21
desc:
keywords: java8
categories: [java]
---

<img src="http://ohwsf74ph.bkt.clouddn.com/image/banner/java8-logo.jpeg">

2014年3月发布的Java 8，有可能是Java版本更新中变化最大的一次。新的Java 8为开发者带来了许多重量级的新特性，包括Lambda表达式，流式数据处理，新的`Optional`类，新的日期和时间API等。这些新特性给Java开发者带来了福音，特别是Lambda表达式的支持，使程序设计更加简化。本篇文章将讨论行为参数化，Lambda表达式，函数式接口等特性。

<!-- more -->

## 行为参数化

在软件开发的过程中，开发人员可能会遇到频繁的需求变更，使他们不断地修改程序以应对这些变化的需求，导致项目进度缓慢甚至项目延期。行为参数化就是一种可以帮助你应对频繁需求变更的开发模式，简单的说，就是预先定义一个代码块而不去执行它，把它当做参数传递给另一个方法，这样，这个方法的行为就被这段代码块参数化了。

为了方便理解，我们通过一个例子来讲解行为参数化的使用。假设我们正在开发一个图书管理系统，需求是要对图书的作者进行过滤，筛选出指定作者的书籍。比较常见的做法就是编写一个方法，把作者当成方法的参数：

```java
public List<Book> filterByAuthor(List<Book> books, String author) {
	List<Book> result = new ArrayList<>();
	for (Book book : books) {
		if (author.equals(book.getAuthor())) {
			result.add(book);
		}
	}
	return result;
}
```

现在客户需要变更需求，添加过滤条件，按照出版社过滤，于是我们不得不再次编写一个方法：

```java
public List<Book> filterByPublisher(List<Book> books, String publisher) {
	List<Book> result = new ArrayList<>();
	for (Book book : books) {
		if (publisher.equals(book.getPublisher())) {
			result.add(book);
		}
	}
	return result;
}
```

两个方法除了名称之外，内部的实现逻辑几乎一模一样，唯一的区别就是`if`判断条件，前者判断的是作者，后者判断的是出版社。如果现在客户又要增加需求，需要按照图书的售价过滤，是不是需要再次将上面的方法复制一遍，将`if`判断条件改为售价？ No! 这种做法违背了DRY（Don’t Repeat Yourself，不要重复自己）原则，而且不利于后期维护，如果需要改变方法内部遍历方式来提高性能，意味着每个`filterByXxx()`方法都需要修改，工作量太大。

一种可行的办法是对过滤的条件做更高层的抽象，过滤的条件无非就是图书的某些属性（比如价格、出版社、出版日期、作者等），可以声明一个接口用于对过滤条件建模：

```java
public interface BookPredicate {
    public boolean test(Book book);
}
```

`BookPredicate`接口只有一个抽象方法`test()`，该方法接受一个`Book`类型参数，返回一个`boolean`值，可以用它来表示图书的不同过滤条件。

接下来我们对之前的过滤方法进行重构，将`filterByXxx()`方法的第二个参数换成上面定义的接口：

```java
public List<Book> filter(List<Book> books, BookPredicate bookPredicate) {
    List<Book> result = new ArrayList<>();
	for (Book book : books) {
		if (bookPredicate.test(book)) {
			result.add(book);
		}
	}
	return result;
}
```

将过滤的条件换成`BookPredicate`的实现类，这里采用了内部类：

```java
// 根据作者过滤
final String author = "张三";
List<Book> result = filter(books, new BookPredicate() {
    @Override
    public boolean test(Book book) {
        return author.equals(book.getAuthor());
    }
});

// 根据图书价格过滤
final double price = 100.00D;
List<Book> result = filter(books, new BookPredicate() {
    @Override
    public boolean test(Book book) {
        return price > book.getPrice();
    }
});
```

重构前后有什么区别？我们将方法中的`if`判断条件换成了`BookPredicate`接口定义的`test()`方法，用于判断是否满足过滤条件，将图书过滤的逻辑交给了`BookPredicate`接口的实现类，而不是在`filter()`方法内部实现过滤，而`BookPredicate`接口又是`filter()`方法的参数。以上的步骤，就是将行为参数化，也就是将图书过滤的行为（`BookPredicate`接口的实现类）当做`filter()`方法的参数。现在，可以删掉所有`filterByXxx()`的方法，只保留`filter()`方法，就算后期数据规模很庞大，需要改变集合的遍历方式来提高性能，只需要在`filter()`方法内部做出相应的修改，而不用去修改其他业务代码。

不过，`BookPredicate`接口只是针对图书的过滤，如果需要对其他对象集合排序（如：用户），又得重新申明一个接口。有一个办法就是可以用Java的泛型对它做进一步的抽象：

```java
public interface Predicate<T> {
    public boolean test(T t);
}
```

现在你可以把`filter()`方法用在任何对象的过滤中。

## Lambda表达式

虽然我们对`filter()`方法进行重构，并抽象了`Predicate`接口作为过滤的条件，但实际上还需要编写很多内部类来实现`Predicate`接口。使用内部类的方式实现`Predicate`接口有很多缺点：首先是代码显得臃肿不堪，可读性差；其次，如果某个局部变量被内部类使用，这个变量必须使用`final`关键字修饰。在Java 8中，使用Lambda表达式可以对内部类进一步简化：

```java
// 根据作者过滤
List<Book> result = filter(books, book -> "张三".equals(book.getAuthor()));

// 根据图书价格过滤
List<Book> result = filter(books, book -> 100 > book.getPrice());
```

使用Lambda仅仅用一行代码就对内部类进行了转化，而且代码变得更加清晰可读。其中`book -> "张三".equals(book.getAuthor())`和`book -> 100 > book.getPrice()`就是我们接下来要研究的Lambda表达式。

### Lambda表达式是什么

Lambda表达式（lambda expression）是一个匿名函数，由数学中的λ演算而得名。在Java 8中可以把Lambda表达式理解为匿名函数，它没有名称，但是有参数列表、函数主体、返回类型等。

Lambda表达式的语法如下：

```java
(parameters) -> { statements; }
```

为什么要使用Lambda表达式？前面你也看到了，在Java中使用内部类显得十分冗长，要编写很多样板代码，Lambda表达式正是为了简化这些步骤出现的，它使代码变得清晰易懂。

### 如何使用Lambda表达式

Lambda表达式是为了简化内部类的，你可以把它当成是内部类的一种简写方式，只要是有内部类的代码块，都可以转化成Lambda表达式：

```java
// Comparator排序
List<Integer> list = Arrays.asList(3, 1, 4, 5, 2);
list.sort(new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1.compareTo(o2);
    }
});

// 使用Lambda表达式简化
list.sort((o1, o2) -> o1.compareTo(o2));
```

```java
// Runnable代码块
Thread thread = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello Man!");
    }
});

// 使用Lambda表达式简化
Thread thread = new Thread(() -> System.out.println("Hello Man!"));
```

可以看出，只要是内部类的代码块，就可以使用Lambda表达式简化，并且简化后的代码清晰易懂。甚至，`Comparator`排序的Lambda表达式还可以进一步简化：

```java
list.sort(Integer::compareTo);
```

这种写法被称为 **方法引用**，方法引用是Lambda表达式的简便写法。如果你的Lambda表达式只是调用这个方法，最好使用名称调用，而不是描述如何调用，这样可以提高代码的可读性。

方法引用使用`::`分隔符，分隔符的前半部分表示引用类型，后面半部分表示引用的方法名称。例如：`Integer::compareTo`表示引用类型为`Integer`，引用名称为`compareTo`的方法。

类似使用方法引用的例子还有打印集合中的元素到控制台中：

```java
list.forEach(System.out::println);
```

## 函数式接口

如果你的好奇心使你翻看`Runnable`接口源代码，你会发现该接口被一个`@FunctionalInterface`的注解修饰，这是Java 8中添加的新注解，用于表示 **函数式接口**。

函数式接口又是什么鬼？在Java 8中，把那些仅有一个抽象方法的接口称为函数式接口。如果一个接口被`@FunctionalInterface`注解标注，表示这个接口被设计成函数式接口，只能有一个抽象方法，如果你添加多个抽象方法，编译时会提示“Multiple non-overriding abstract methods found in interface XXX”之类的错误。

函数式方法又能做什么？Java8允许你以Lambda表达式的方式为函数式接口提供实现，通俗的说，你可以将整个Lambda表达式作为接口的实现类。

除了`Runnable`之外，Java 8中内置了许多函数式接口供开发者使用，这些接口位于`java.util.function`包中，我们之前使用的`Predicate`接口，已经被包含在这个包内，他们分别为`Predicate`、`Consumer`和`Function`，由于我们已经在之前的图书过滤的例子中介绍了`Predicate`的用法，所以接下来主要介绍`Consumer`和`Function`的用法。

### `Consumer`

`java.util.function.Consumer<T>`定义了一个名叫`accept()`的抽象方法，它接受泛型`T`的对象，没有返回（`void`）。如果你需要访问类型`T`的对象，并对其执行某些操作，就可以使用这个接口。比如，你可以用它来创建一个`forEach()`方法，接受一个集合，并对集合中每个元素执行操作：

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

public static <T> void forEach(List<T> list, Consumer<T> consumer) {
    for(T t: list){
        consumer.accept(t);
    }
}

public static void main(String[] args) {
    List<String> list = Arrays.asList("A", "B", "C", "D");
    forEach(list, str -> System.out.println(str));
    // 也可以写成
    forEach(list, System.out::println);
}
```

### `Function`

`java.util.function.Function<T, R>`接口定义了一个叫作`apply()`的方法，它接受一个泛型`T`的对象，并返回一个泛型`R`的对象。如果你需要定义一个Lambda，将输入对象的信息映射到输出，就可以使用这个接口。比如，我们需要计算一个图书集合中每本书的作者名称有几个汉字（假设这些书的作者都是中国人）：

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
    List<R> result = new ArrayList<>();
    for(T s: list){
        result.add(f.apply(s));
    }
    return result;
}

public static void main(String[] args) {
    List<Book> books = Arrays.asList(
        new Book("张三", 99.00D),
        new Book("李四", 59.00D),
        new Book("王老五", 59.00D)
    );
    List<Integer> results = map(books, book -> book.getAuthor().length());
}
```

现在，你应该对Lambda表达式有一个初步的了解了，并且，你可以使用Lambda表达式来重构你的代码，提高代码可读性；使用行为参数化来设计你的程序，让程序更灵活。在下一篇文章将会介绍Java 8的另一个特性——流式数据处理。
