---
title: "实验：在 Visual C++ 中使用 mruby"
categories:
  - Ruby
  - C
tags:
  - Visual C++
  - mruby
---

Ruby (MRI, Matz's Ruby Interpreter) 与 MSVC 的兼容性不好，想要在 Visual C++ 中使用 Ruby 作为脚本语言来运行，可以选择 [mruby](http://mruby.org/)。本篇文章介绍如何在 Windows 上编译 mruby，并在 Visual C++ 项目中添加 mruby 的标头和库文件。

mruby 是 Ruby 语言的轻量级实现，由 Ruby 的作者松本行弘主持开发，瞄准的是 lua 的生态位，也就是让 Ruby 可以被更容易地链接和嵌入到 C/C++ 应用程序中运行。著名游戏[《尼尔：机械纪元》](https://store.steampowered.com/app/524220/NieRAutomata/)就是使用了 mruby 作为脚本语言。

实验环境：

- Windows 11 23H2 64 位
- Visual Studio 2022
- MSVC v143

mruby 版本 `3.3.0`，mruby 的源代码可以在 [mruby.org/downloads](https://mruby.org/downloads/) 页面或者 Github 仓库 [mruby/mruby](https://github.com/mruby/mruby) 下载。

编译 mruby 前需要安装 Ruby，因为 mruby 的构建系统是用 Ruby 写的。在 Windows 上安装 Ruby 可以使用 [RubyInstaller for Windows](https://rubyinstaller.org/)，推荐下载安装带 Devkit 的版本，它会自动安装一份 Ruby 需要的 MSYS2 环境。我安装的 Ruby 版本是 `3.3.6`。

下载 mruby 的源代码后，解压到一个工作目录。我将 mruby 的源代码放在了 `D:\Workspace\ruby\mruby-3.3.0`，下面的过程请你对应自己的实际目录。

打开 `开始` > `Visual Studio 2022` > `x64 Native Tools Command Prompt for VS 2022`，这个命令行就是我们编译 mruby 的环境。

执行以下命令：

```bat
:: 进入驱动器
D:

:: 进入源代码根目录
cd D:\Workspace\ruby\mruby-3.3.0

:: 编译
rake

:: 测试
rake test

:: 安装（默认到当前驱动器的 \usr\local 目录）
rake install
```

编译完成后，我们进到 `D:\usr\local` 下就能看到 mruby 的相关文件了。

```
\usr\local\
|-- bin\
|   |-- mrbc.exe           # 编译器
|   |-- mruby.exe          # 解释器
|   |-- mruby-config.bat   # 配置工具
|   \-- ...
|-- include\
|   |-- mruby.h            # 标头文件
|   |-- mrbconf.h          # 配置信息标头
|   \-- mruby\...          # 更多标头文件
\-- lib\
    |-- libmruby.lib       # 库文件
    \-- ...
```

进入到 `D:\usr\local\bin` 目录中，使用 `mruby-config.bat` 配置工具来获取配置选项：

```bat
:: 进入 bin 目录
cd D:\usr\local\bin

:: 获取编译器选项
.\mruby-config.bat --cflags
:: 以 /I 开头的参数就是我们需要配置的 INCLUDE 包含路径，我的输出类似：
::   /I"D:\usr\local\include"
::   /I"D:\usr\local\include\mruby\gems\mruby-time\include"
::   /I"D:\usr\local\include\mruby\gems\mruby-io\include"

:: 获取链接器选项
.\mruby-config.bat --libs
:: 每个库以空格分割，它是我们需要配置的附加依赖项，我的输出类似：
::   libmruby.lib
::   ws2_32.lib
::   wsock32.lib
```

打开 `Visual Studio 2022`，创建一个普通的命令行 C++ 项目，再将源代码文件改为我们的实验代码：

```c
#include <stdio.h>
#include <mruby.h>
#include <mruby/compile.h>

int main(int argc, char *argv[]) {
    puts("Hello C!");

    // 打开 mruby 执行上下文
    mrb_state *mrb = mrb_open();

    mrb_load_string(mrb, "puts 'Hello mruby!'");

    // 关闭 mruby 执行上下文
    mrb_close(mrb);

    return 0;
}
```

在 `项目` > `属性` 中配置 `所有配置` 和 `x64` 平台：

1. `配置属性` > `VC++ 目录`，为 `包含目录` 添加 `mruby-config.bat --cflags` 输出的 `/I` 参数目录
2. `配置属性` > `VC++ 目录`，为 `库目录` 添加 `libmruby.lib` 文件所在的目录，我这里就是 `D:\usr\local\lib`
   ![pic1](/assets/images/posts/2024-11-26-using-mruby-in-visual-c++/pic1.png)
3. `配置属性` > `链接器` > `输入`，为 `附加依赖项` 添加 `mruby-config.bat --libs` 输出的库文件名，我这里就是 `libmruby.lib` `ws2_32.lib` `wsock32.lib`
   ![pic2](/assets/images/posts/2024-11-26-using-mruby-in-visual-c++/pic2.png)

生成项目，点击 ▶️ 图标，就能看到输出：

```
Hello C!
Hello mruby!
```
