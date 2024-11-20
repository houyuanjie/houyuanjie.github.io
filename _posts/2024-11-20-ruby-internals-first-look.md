---
title: "Ruby 内部初探"
categories:
  - Ruby
tags:
  - Prism
  - RubyVM
---

Ruby 的执行流程与 JVM 类似，它会先将源码编译成字节码，再由 RubyVM 执行。我们利用 Ruby 的解析器 `Prism` 和 `RubyVM::InstructionSequence` 两个工具对 Ruby 的执行流程进行初探。

Prism 是 Ruby 新实现的一个解析器，它将从 3.4 版本开始代替 parse.y 成为内部默认的解析器。同时 Prism 也是一个可用的 Gem，在本篇中我们就用它来代替旧的 Ripper 工具来解析 Ruby 源码。

```ruby
require 'prism'

Prism.lex   # str -> tokens
Prism.parse # str -> ast
```

RubyVM::InstructionSequence 即 RubyVM 的指令序列，它提供了探索 RubyVM 内部工作原理的一些方法。

```ruby
RubyVM::InstructionSequence.compile # str -> inseq
inseq.eval   # -> obj
inseq.disasm # -> str
```

需要注意，在不同的版本中，解析、编译的结果可能会有不同，我使用的版本是 `3.3.6`，可以使用 `ruby --version` 查看你的 Ruby 版本。

我们的实验代码如下：

```ruby
require 'prism'
require 'pp'

code = <<~RUBY
  a = 1
  b = 2
  puts a + b + 3
RUBY

pp Prism.lex(code)
pp Prism.parse(code)
pp RubyVM::InstructionSequence.compile(code).disasm

eval code
```

首先我们来看分词（lex），省略掉我们不关心的信息后，分词结果就像下面这样：

```ruby
  [[IDENTIFIER(1,0)-(1,1)("a"), 32],
   [EQUAL(1,2)-(1,3)("="), 1],
   [INTEGER(1,4)-(1,5)("1"), 2],
   [NEWLINE(1,5)-(2,0)("\n"), 1],

   [IDENTIFIER(2,0)-(2,1)("b"), 32],
   [EQUAL(2,2)-(2,3)("="), 1],
   [INTEGER(2,4)-(2,5)("2"), 2],
   [NEWLINE(2,5)-(3,0)("\n"), 1],

   [IDENTIFIER(3,0)-(3,4)("puts"), 32],
   [IDENTIFIER(3,5)-(3,6)("a"), 1026],
   [PLUS(3,7)-(3,8)("+"), 1],
   [IDENTIFIER(3,9)-(3,10)("b"), 1026],
   [PLUS(3,11)-(3,12)("+"), 1],
   [INTEGER(3,13)-(3,14)("3"), 2],
   [NEWLINE(3,14)-(4,0)("\n"), 1],

   [EOF(4,0)-(4,0)(""), 1]]
```

- `a` `b` `puts` 被识别为了标识符 `IDENTIFIER`
- `1` `2` `3` 被识别为了整数 `INTEGER`
- `=` `+` 被识别为了运算符 `EQUAL` `PLUS`

解析（parse）后的语法树非常大，我就不全贴出来了，它的结构大概类似这样：

```ruby
@ ProgramNode
  statements:
    @ StatementsNode
      body:
        @ LocalVariableWriteNode :a
          @ IntegerNode 1
        @ LocalVariableWriteNode :b
          @ IntegerNode 2
        @ CallNode :puts
          arguments:
            @ ArgumentsNode
              arguments:
                @ CallNode :+
                  receiver:
                    @ CallNode :+
                      receiver:
                        @ LocalVariableReadNode :a
                      arguments:
                        @ ArgumentsNode
                          arguments:
                            @ LocalVariableReadNode :b
                  arguments:
                    @ ArgumentsNode
                      arguments:
                        @ IntegerNode 3

```

比较有意思的是 `puts` 的参数 `a + b + 3` 的部分

它的表达式类似 `(:+ (:+ a b) 3)`，即 `a` 与 `b` 相加，结果再与 `3` 相加，在下面的字节码中，我们会看到编译后的指令序列，它的顺序与这个树是一致的。

为了看得更清楚，我标出来了指令执行时的栈状态

```javascript
"== disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(3,14)>\n";
// 本地表开始
"local table (size: 2, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])\n";
"[ 2] a@0        [ 1] b@1\n";
// 本地表结束

// 指令序列开始
{ stack: [] };
"0000 putobject_INT2FIX_1_                                             (   1)[Li]\n";
{ stack: [1] };
"0001 setlocal_WC_0                          a@0\n";
{ stack: [], a: 1 };
"0003 putobject                              2                         (   2)[Li]\n";
{ stack: [2], a: 1 };
"0005 setlocal_WC_0                          b@1\n";
{ stack: [], a: 1, b: 2 };
"0007 putself                                                          (   3)[Li]\n";
{ stack: [self], a: 1, b: 2 };
"0008 getlocal_WC_0                          a@0\n";
{ stack: [self, 1], a: 1, b: 2 };
"0010 getlocal_WC_0                          b@1\n";
{ stack: [self, 1, 2], a: 1, b: 2 };
"0012 opt_plus                               <calldata!mid:+, argc:1, ARGS_SIMPLE>[CcCr]\n";
{ stack: [3], a: 1, b: 2 };
"0014 putobject                              3\n";
{ stack: [3, 3], a: 1, b: 2 };
"0016 opt_plus                               <calldata!mid:+, argc:1, ARGS_SIMPLE>[CcCr]\n";
{ stack: [6], a: 1, b: 2 };
"0018 opt_send_without_block                 <calldata!mid:puts, argc:1, FCALL|ARGS_SIMPLE>\n";
{ stack: [], a: 1, b: 2 };
"0020 leave\n";
// 指令序列结束
```

可以看到，Ruby 的字节码指令与 JVM 指令非常相似，因为它们都是基于栈的虚拟机，处理逻辑是一样的。

与 Java 不同的是 Ruby 把解析和编译的过程隐藏了起来，我们直接指定的是源文件，解析和编译的过程在 Ruby 执行的内部。

Ruby 的优势在于简化的开发流程，我们无需关心编译过程，直接编写源代码即可。这种简化提高了开发效率，适合快速迭代和原型开发。

Java 提供了更高的自由度，开发者可以选择多种 JVM 语言，并且通过稳定的字节码格式实现跨语言的兼容性。当然，这也要求开发者对编译过程和字节码有一定的了解，增加了开发的复杂性和学习曲线。

所以技术是一种权衡，选择哪种技术时要考虑具体的需求和应用场景。
