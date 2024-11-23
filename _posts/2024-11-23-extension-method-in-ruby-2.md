---
title: "Ruby 中的扩展方法 2"
categories:
  - Ruby
tags:
  - Syntax
---

在之前的一篇文章 [Ruby 中的扩展方法](/2024/11/19/extension-method-in-ruby) 中，我们对比 Scala 扩展方法，介绍了一种在 Ruby 中给现有类添加新方法的方式，称为“打开类”，这篇文章将继续这个主题，并介绍 Ruby 中的单例类（Singleton Class）。

在 Ruby 中，类是方法的集合，同时 Ruby 是一门完全面向对象语言，一个类也是一个对象，所以你能想象，当我们说“打开一个类，并在其中添加方法”时，我们实际上在做的是”给集合中添加元素“这种操作。

我们提到的打开类的语法：

```ruby
class Integer
  def is_even? = self % 2 == 0
  def is_odd?  = self % 2 != 0
end
```

就可以理解为：

```ruby
# 不是有效的 Ruby 代码，仅供理解
# 注释掉的行是有效的 Ruby 代码

Integer['is_even?'] = { self % 2 == 0 }
# Integer.define_method(:is_even?) { self % 2 == 0 }
Integer['is_odd?']  = { self % 2 != 0 }
# Integer.define_method(:is_odd?)  { self % 2 != 0 }
```

这将为 `Integer` 类的所有实例添加 `is_even?` 和 `is_odd?` 方法。

那么如何给一个类添加类方法呢？也就是类似我们 Scala 中的定义在伴生对象里的方法。

```ruby
class Greeter; end
```

以这个 `Greeter` 类为例，我们需要打开它的单例类（Singleton Class），然后在单例类中添加方法：

```ruby
class Greeter
  # 打开单例类
  class << self
    def hello = puts "Hello!"
  end
end

Greeter.hello # Hello!
```

我们定义的 `Greeter` 类在某种程度上可以理解为是自己单例类的一个实例。你能想象，它类似于：

```ruby
# 不是有效的 Ruby 代码，仅供理解
# 注释掉的行是有效的 Ruby 代码

GreeterSingleton['hello'] = { puts "Hello!" }
Greeter = GreeterSingleton.new
# Greeter.singleton_class.define_method(:hello) { puts 'Hello!' }
# Greeter.define_singleton_method(:hello) { puts 'Hello!' }
```

这种 `class << self` 语法看起来实际可能有点奇怪，初学者可能会觉得它和列表的 `<<` `append` 操作符会产生混淆，其实当你理解了“类是方法的集合”“类是对象”时，就能明白 Ruby 设计者选择这样的一个语法是非常贴切的一种表达。

当然 Ruby 也提供了另外一种写法：

```ruby
class Greeter
  def self.hello = puts "Hello!"
end
```

这种写法直接将 `self.` 加在了类方法名的前面，我认为这是一种更直观的写法，你可以根据自己的喜好选择语法。
