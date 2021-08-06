---
title: Java 8新特性（三）：Optional类
date: 2017-01-03
desc:
keywords: java8
categories: [java]
---

<img src="https://raw.githubusercontent.com/lw900925/blog-asset/master/images/banner/java8-logo.jpeg">

在上一篇介绍Stream流式数据处理的文章中提到了`Optional`类，这是Java 8新增的一个类，用以解决程序中常见的`NullPointerException`异常问题。本篇文章将详细介绍`Optional`类，以及如何用它消除代码中的`null`检查。

<!-- more -->

## 避免使用`null`检查

作为Java开发人员，几乎所有人都遇到过`NullPointerException`异常，大多数人遇到`NullPointerException`异常时都会在异常出现的地方加上`if`代码块来判断值不为空，比如下面的代码：

```java
public void bindUserToRole(User user) {
    if (user != null) {
        String roleId = user.getRoleId();
        if (roleId != null) {
            Role role = roleDao.findOne(roleId);
            if (role != null) {
                role.setUserId(user.getUserId());
                roleDao.save(role);
            }
        }
    }
}
```

这是比较普遍的做法，为了避免出现`NullPointerException`异常，手动对可能为`null`值进行了处理，只不过代码可读性就没那么好，业务逻辑都被淹没在`if`逻辑判断中，重构一下看起来会好一点：

```java
public String bindUserToRole(User user) {
    if (user == null) {
        return;
    }

    String roleId = user.getRoleId();
    if (roleId == null) {
        return;
    }

    Role = roleDao.findOne(roleId);
    if (role != null) {
        role.setUserId(user.getUserId());
        roleDao.save(role);
    }
}
```

上面的代码避免了深层的`if`语句嵌套，但本质上是一样的，方法内有三个不同的返回点，出错后调试也不容易，你都不知道是那个值导致了`NullPointerException`异常。

基于上面的原因，Java 8中引入了一个新的类`Optional`，用以避免使用`null`值引发的种种问题。

## `Optional`类

`java.util.Optional<T>`类是一个封装了`Optional`值的容器对象，`Optional`值可以为`null`，如果值存在，调用`isPresent()`方法返回`true`，调用`get()`方法可以获取值。

### 创建`Optional`对象

`Optional`类提供类三个方法用于实例化一个`Optional`对象，它们分别为`empty()`、`of()`、`ofNullable()`，这三个方法都是静态方法，可以直接调用。

---

`empty()`方法用于创建一个没有值的`Optional`对象：

```java
Optional<String> emptyOpt = Optional.empty();
```

`empty()`方法创建的对象没有值，如果对`emptyOpt`变量调用`isPresent()`方法会返回`false`，调用`get()`方法抛出`NullPointerException`异常。

---

`of()`方法使用一个非空的值创建`Optional`对象：

```java
String str = "Hello World";
Optional<String> notNullOpt = Optional.of(str);
```

---

`ofNullable()`方法接收一个可以为`null`的值：

```java
Optional<String> nullableOpt = Optional.ofNullable(str);
```

如果`str`的值为`null`，得到的`nullableOpt`是一个没有值的`Optional`对象。

### 提取`Optional`对象中的值

如果我们要获取`User`对象中的`roleId`属性值，常见的方式是直接获取：

```java
String roleId = null;
if (user != null) {
    roleId = user.getRoleId();
}
```

使用`Optional`中提供的`map()`方法可以以更简单的方式实现：

```java
Optional<User> userOpt = Optional.ofNullable(user);
Optional<String> roleIdOpt = userOpt.map(User::getRoleId);
```

`map()`方法与Stream API中的`map()`一样，类似于映射操作，将原始类型映射为一个新的类型。

### 使用`orElse()`方法获取值

`Optional`类还包含其他方法用于获取值，这些方法分别为：

- `orElse()`：如果有值就返回，否则返回一个给定的值作为默认值；
- `orElseGet()`：与`orElse()`方法作用类似，区别在于生成默认值的方式不同。该方法接受一个`Supplier<? extends T>`函数式接口参数，用于生成默认值；
- `orElseThrow()`：与前面介绍的`get()`方法类似，当值为`null`时调用这两个方法都会抛出`NullPointerException`异常，区别在于该方法可以指定抛出的异常类型。

下面来看看这三个方法的具体用法：

```java
String str = "Hello World";
Optional<String> strOpt = Optional.of(str);
String orElseResult = strOpt.orElse("Hello Shanghai");
String orElseGet = strOpt.orElseGet(() -> "Hello Shanghai");
String orElseThrow = strOpt.orElseThrow(() -> new IllegalArgumentException("Argument 'str' cannot be null or blank."));
```

此外，`Optional`类还提供了一个`ifPresent()`方法，该方法接收一个`Consumer<? super T>`函数式接口，可以对值进行一些操作：

```java
Optional<String> strOpt = Optional.of("Hello World");
strOpt.ifPresent(System.out::println);
```

### 使用`filter()`方法过滤

`filter()`方法可用于判断`Optional`对象是否满足给定条件，一般用于条件过滤：

```java
Optional<String> optional = Optional.of("lw900925@163.com");
optional = optional.filter(str -> str.contains("164"));
```

在上面的代码中，如果`filter()`方法中的Lambda表达式成立，`filter()`方法会返回当前`Optional`对象值，否则，返回一个值为空的`Optional`对象。

## 如何正确使用`Optional`

我发现好像很多人（别说还真的很多）没有掌握`Optional`的正确使用方法，比如我一个同事喜欢写这样的`Optional`代码：

```java
Optional<User> userOpt = Optional.ofNullable(user);
if (userOpt.isPresent()) {
    User user = userOpt.get();
    // do something...
} else {
    // do something...
}

```

说实话，这趟的写法跟传统的`if`语句判断空值没有任何区别，没有起到`Optional`的正真作用（中枪的同学举手）：

```java
if (user != null) {
    // do something...
} else {
    // do something...
}
```

所以，当我们从之前版本切换到Java 8的时候，不应该还按照以前的思维方式处理`null`值，Java 8提倡函数式编程，新增的许多API都可以用函数式编程表示，`Optional`类也是其中之一。这里有几条关于`Optional`使用的建议：

1. 尽量避免在程序中直接调用`Optional`对象的`get()`和`isPresent()`方法；
2. 避免使用`Optional`类型声明实体类的属性；

第一条建议中直接调用`get()`方法是很危险的做法，如果`Optional`的值为空，那么毫无疑问会抛出`NullPointerException`异常，而为了调用`get()`方法而使用`isPresent()`方法作为空值检查，这种做法与传统的用`if`语句块做空值检查没有任何区别。

第二条建议避免使用`Optional`作为实体类的属性，它在设计的时候就没有考虑过用来作为类的属性，可以查看`Optional`的源代码，你会发现它没有实现`java.io.Serializable`接口，也就是说如果你用到一些orm框架的二级缓存，使用`Optional`作为实体类的属性没法被序列化。

下面我们通过一些例子讲解`Optional`的正确用法：

### 正确创建`Optional`对象

上面提到创建`Optional`对象有三个方法，`empty()`方法比较简单，没什么特别要说明的。主要是`of()`和`ofNullable()`方法。当你很确定一个对象不可能为`null`的时候，应该使用`of()`方法，否则，尽可能使用`ofNullable()`方法，比如：

```java
public static void method(Role role) {
    // 当Optional的值通过常量获得或者通过关键字new初始化，可以直接使用of()方法
    Optional<String> strOpt = Optional.of("Hello World");
    Optional<User> userOpt = Optional.of(new User());

    // 方法参数中role值不确定是否为null，使用ofNullable()方法创建
    Optional<Role> roleOpt = Optional.ofNullable(role);
}
```

### `orElse()`方法的使用

```java
return str != null ? str : "Hello World"
```

上面的代码表示判断字符串`str`是否为空，不为空就返回，否则，返回一个常量。使用`Optional`类可以表示为：

```java
return strOpt.orElse("Hello World")
```

### 简化`if-else`

```java
User user = ...
if (user != null) {
    String userName = user.getUserName();
    if (userName != null) {
        return userName.toUpperCase();
    } else {
        return null;
    }
} else {
    return null;
}
```

上面的代码可以简化成：

```java
User user = ...
Optional<User> userOpt = Optional.ofNullable(user);

return userOpt.map(User::getUserName)
            .map(String::toUpperCase)
            .orElse(null);
```


再比如判断用户名不能重复的逻辑，根据用户名从数据库中查询一个用户，如果不为null就抛出异常告诉前端用户已存在：

```java
User existUser = userDAO.findById(user.getUsername());
if (existUser != null) {
    throw new AppException("用户名已存在");
}

```

可以简写为：

```java
User user = userDAO.findById(user.getId());
Optional.ofNullable(user).ifPresent(it -> throw new AppException("用户名已存在"));
```

总结一下，新的`Optional`类让我们可以以函数式编程的方式处理`null`值而不用嵌套很多`if-else`逻辑。下一篇文章将介绍Java 8中新添加的日期API。
