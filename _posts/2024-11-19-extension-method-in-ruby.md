---
title: "Ruby 中的扩展方法"
categories:
  - Ruby
tags:
  - Syntax
---

在 Ruby 中如何给一个已经存在的类添加新的方法？

例如，我们希望给整数添加 `is_even?` 和 `is_odd?` 方法，用来判断一个整数是否为偶数或者奇数。

```ruby
3.is_even? # false
3.is_odd?  # true
```

在 Scala 中，我们可以使用 `extension` 关键字来实现，类似这样：

```scala
extension (n: Int)
  def isEven = n % 2 == 0
  def isOdd  = n % 2 != 0

3.isEven // false
3.isOdd  // true
```

在 Scala 2 中，我们使用 `implicit class` 来实现：

```scala
implicit class IntOps(n: Int) {
  def isEven = n % 2 == 0
  def isOdd  = n % 2 != 0
}
```

编译时的代码类似这样：

```scala
// 3.isEven
IntOps(3).isEven
// 3.isOdd
IntOps(3).isOdd
```

可以看到，Scala 的实现原理是对原类进行包装，再通过包装类扩展方法。

与 Scala 不同，在 Ruby 中，我们不需要对类进行包装，而是可以直接扩展已有类：

```ruby
class Integer
  def is_even? = self % 2 == 0
  def is_odd?  = self % 2 != 0
end
```

> 方法体中的 `self` 关键字指代当前调用该方法的对象实例。

> 方法名中的 `?` 表示这个方法返回一个布尔值

在 Ruby 社区中通常将这种手法称作打开类（Open Class），或者猴子补丁（Monkey Patch），是一种在运行时修改类的行为的一种方法。

运行时的代码类似这样：

```ruby
# 3.is_even?
3.send(:is_even?)
# 3.is_odd?
3.send(:is_odd?)
```
