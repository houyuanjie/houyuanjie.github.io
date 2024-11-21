---
title: 实验：在 C 中运行 Ruby 脚本
categories:
  - C
  - Ruby
tags:
  - GCC
  - VSCode
---

很多大型项目引入脚本语言来提高灵活性。例如当在 IDEA 中调试时进入断点后，IDE 允许我们写一段 Groovy 脚本来检查上下文；在 Nginx 中，编写 Lua 脚本来扩展功能；运行在浏览器中的 JavaScript 能对页面的功能进行增强。本篇文章我们来尝试一下在 C 中运行 Ruby 脚本。

本片文章参考了 [The Definitive Guide to Ruby's C API](https://silverhammermba.github.io/emberb/)，原文的仓库在 Github [silverhammermba/emberb](https://github.com/silverhammermba/emberb) 可以找到。那篇文章很详细，想要了解有关 Ruby 的 C API 的内容，请阅读那篇文章，我在这篇文章中的关注点将放在 GCC 和 VSCode 的配置上。

Ruby 本身使用 C 语言编写，所以在 C 中调用 Ruby 的 API 是非常简单的，简单到只需要包含头文件 `ruby.h` 即可。但对程序员来说，往往最困难的就是配置开发环境，接下来请跟我一起来做这个实验。

要在 Ubuntu Linux 上开始实验，我们需要先运行下面的命令来安装依赖：

```sh
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install build-essential gdb ruby ruby-dev pkgconf
```

编写这篇文章时 WSL 默认的 Ubuntu 版本是 `24.04.1 LTS`，其中包含的 Ruby 版本是 `3.2`，GCC 版本是 `13.2`。

安装完成依赖后我们开始创建并打开一个项目：

```sh
# 在用户的家目录中
$ mkdir workspace/c-projects/ruby-in-c
$ cd workspace/c-projects/ruby-in-c
$ code .
```

在 VSCode 中打开项目后，我们创建一个名为 `main.c` 的文件，并输入我们的实验代码：

```c
// main.c
#include <stdio.h>
#include <ruby.h>

int main(int argc, char *argv[])
{
    ruby_init();

    printf("Hello C!\n");
    rb_eval_string("puts 'Hello Ruby!'");

    return ruby_cleanup(0);
}
```

其中：

- `ruby_init()` 用于初始化 RubyVM，它是 Ruby 代码执行的环境
- `rb_eval_string()` 用于快速执行一段 Ruby 代码
- `ruby_cleanup()` 用于清理并关闭 RubyVM，参数是正常关闭后返回的默认值

保存文件后，我们来配置 VSCode。

首先 `IntelliSense` 将提示我们找不到头文件 `ruby.h`，将鼠标放置在标红的 `#include <ruby.h>` 这一行，然后点击添加 `ruby-3.2.0` 和 `x86_64-linux-gnu/ruby-3.2.0` 到 `IncludePath` 中，这将在 `.vscode` 目录下生成一个 `c_cpp_properties.json` 文件，内容大致如下：

```json
{
  "configurations": [
    {
      "name": "Linux",
      "includePath": [
        "${workspaceFolder}/**",
        /* IntelliSense 需要下面两行配置来查找 Ruby 的头文件 */
        "/usr/include/ruby-3.2.0",
        "/usr/include/x86_64-linux-gnu/ruby-3.2.0"
      ],
      "defines": [],
      "compilerPath": "/usr/bin/gcc",
      "cStandard": "c17",
      "cppStandard": "gnu++17",
      "intelliSenseMode": "linux-gcc-x64"
    }
  ],
  "version": 4
}
```

接着我们来配置 VSCode 的生成和调试任务。

点击编辑器右上角的 ⚙️ 图标，VSCode 将自动检测系统中安装的 `gcc` 和 `gdb`，选择 `C/C++: gcc 生成和调试活动文件`，VSCode 将自动生成 `launch.json` 和 `tasks.json` 文件。

![pic1](/assets/images/posts/2024-11-21-running-ruby-script-in-c/pic1.jpg)

生成的 `launch.json` 文件内容大致如下：

```json
{
  "configurations": [
    {
      "name": "C/C++: gcc 生成和调试活动文件",
      "type": "cppdbg",
      "request": "launch",
      "program": "${fileDirname}/${fileBasenameNoExtension}",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${fileDirname}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "setupCommands": [
        {
          "description": "为 gdb 启用整齐打印",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        },
        {
          "description": "将反汇编风格设置为 Intel",
          "text": "-gdb-set disassembly-flavor intel",
          "ignoreFailures": true
        }
      ],
      "preLaunchTask": "C/C++: gcc 生成活动文件",
      "miDebuggerPath": "/usr/bin/gdb"
    }
  ],
  "version": "2.0.0"
}
```

生成的 `tasks.json` 文件内容大致如下：

```json
{
  "tasks": [
    {
      "type": "cppbuild",
      "label": "C/C++: gcc 生成活动文件",
      "command": "/usr/bin/gcc",
      "args": [
        "-fdiagnostics-color=always",
        "-g",
        "${file}",
        "-o",
        "${fileDirname}/${fileBasenameNoExtension}"
        /* start Ruby */

        /* end Ruby */
      ],
      "options": {
        "cwd": "${fileDirname}"
      },
      "problemMatcher": ["$gcc"],
      "group": "build",
      "detail": "调试器生成的任务。"
    }
  ],
  "version": "2.0.0"
}
```

我们需要在 `start Ruby` 和 `end Ruby` 注释之间为 GCC 添加 处理 Ruby 头文件的参数。

打开终端并运行：

```sh
$ pkg-config --cflags --libs ruby
```

![pic2](/assets/images/posts/2024-11-21-running-ruby-script-in-c/pic2.jpg)

将 `pkgconf` 的输出添加到我们之前在 `tasks.json` 中预留的位置。注意，输出是一个空格分隔的列表，我们要添加成多条参数。

最终的 `tasks.json` 文件内容如下：

```json
{
  "tasks": [
    {
      "type": "cppbuild",
      "label": "C/C++: gcc 生成活动文件",
      "command": "/usr/bin/gcc",
      "args": [
        "-fdiagnostics-color=always",
        "-g",
        "${file}",
        "-o",
        "${fileDirname}/${fileBasenameNoExtension}",
        /* start Ruby */
        "-I/usr/include/x86_64-linux-gnu/ruby-3.2.0", // Ruby 编译配置信息头文件
        "-I/usr/include/ruby-3.2.0", // Ruby 头文件
        "-lruby-3.2", // 链接 ruby
        "-lm", // 链接 math
        "-lpthread" // 链接 POSIX thread
        /* end Ruby */
      ],
      "options": {
        "cwd": "${fileDirname}"
      },
      "problemMatcher": ["$gcc"],
      "group": "build",
      "detail": "调试器生成的任务。"
    }
  ],
  "version": "2.0.0"
}
```

现在，点击 `main.c` 文件编辑器右上角的 ▶️ 图标，或者在 `运行和调试` 面板中选择这个任务，点击开始，即可运行代码，输出结果如下：

```
Hello C!
Hello Ruby!
```
