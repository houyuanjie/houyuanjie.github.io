---
title: "使用 Stow 管理 Linux 上的包"
categories:
  - Linux
tags:
  - GNU Stow
---

我一直有一个疑惑：当我 `make install` 之后，怎么删掉这个包？

随着我对 Linux 的了解逐渐加深，我有了一个构想：如果我能记录下 `make install` 时安装到 `/usr/local` 目录下的所有文件，我就能在不需要它们时将它们一起删掉

后来我学到了软链接，我想到了更好的办法：我可以在安装时设置将软件的 `--prefix` 到一个独立的目录，然后为这些文件创建到 `/usr/local` 目录的软链接。而且这似乎是一种非常符合 Unix 哲学的想法，这时我发现了 [GNU Stow](https://www.gnu.org/software/stow/)

# 1. 安装 Stow

要在 Ubuntu 上安装 Stow，运行下面这条命令：

```sh
sudo apt install stow
```

# 2. 使用方式

运行下面这条命令可以查看帮助信息：

```sh
stow --help
```

输出的帮助信息如下：

```
stow (GNU Stow) 版本 2.3.1

概要:

    stow [选项 ...] [-D|-S|-R] 软件包名 ... [-D|-S|-R] 软件包名 ...

选项:

    -d 目录, --dir=目录      设置 stow 目录为指定目录（默认为当前目录）
    -t 目录, --target=目录   设置目标目录为指定目录（默认为 stow 目录的父目录）

    -S, --stow            存储(stow)后续列出的软件包
    -D, --delete          删除(unstow)后续列出的软件包
    -R, --restow          重新存储（相当于先执行 stow -D 再执行 stow -S）

    --ignore=正则表达式    忽略匹配此 Perl 正则表达式结尾的文件
    --defer=正则表达式     如果文件已被其他软件包存储，则推迟存储以该正则表达式开头的文件
    --override=正则表达式  如果文件已被其他软件包存储，仍强制存储以该正则表达式开头的文件
    --adopt               （谨慎使用！）将目标目录中的现有文件整合到 stow 软件包中。使用前请阅读文档。
    -p, --compat          使用旧算法进行卸载(unstow)

    -n, --no, --simulate  仅模拟操作，不实际修改文件系统
    -v, --verbose[=N]     增加详细输出级别（级别范围为 0 到 5；
                            -v 或 --verbose 增加 1 级；--verbose=N 直接设置级别）
    -V, --version         显示 stow 版本号
    -h, --help            显示本帮助信息

错误报告请发送至: bug-stow@gnu.org
Stow 主页: <http://www.gnu.org/software/stow/>
GNU 软件使用帮助: <http://www.gnu.org/gethelp/>
```

# 3. 示例

以在 Linux 上编译安装 Nginx (1.27.4) 为例

首先下载 Nginx 的源码包

```sh
mkdir -p ~/Downloads
cd ~/Downloads

curl -O https://nginx.org/download/nginx-1.27.4.tar.gz

tar -xzvf nginx-1.27.4.tar.gz
```

然后安装必要的依赖

```sh
sudo apt update && sudo apt upgrade
sudo apt install build-essential libpcre3 libpcre3-dev zlib1g-dev libssl-dev
```

接着进行配置和编译

注意我们将安装目录设置为了 `/usr/local/stow/nginx-1.27.4`

```sh
cd nginx-1.27.4

./configure --prefix=/usr/local/stow/nginx-1.27.4
make
```

最后进行安装

```sh
sudo make install
```

安装完成后我们去看一下安装了哪些文件

```sh
cd /usr/local/stow

tree nginx-1.27.4
```

输出的目录结构如下：

```
nginx-1.27.4/
├── conf
│   ├── fastcgi.conf
│   ├── fastcgi.conf.default
│   ├── fastcgi_params
│   ├── fastcgi_params.default
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types
│   ├── mime.types.default
│   ├── nginx.conf
│   ├── nginx.conf.default
│   ├── scgi_params
│   ├── scgi_params.default
│   ├── uwsgi_params
│   ├── uwsgi_params.default
│   └── win-utf
├── html
│   ├── 50x.html
│   └── index.html
├── logs
└── sbin
    └── nginx

5 directories, 18 files
```

可以看到，一个标准的 Nginx 包结构已经安装到了 `/usr/local/stow/nginx-1.27.4` 目录下

接下来就到了 Stow 起作用的时候了，非常简单，我们在 `/usr/local/stow` 目录下运行这条命令

```sh
sudo stow nginx-1.27.4
```

我们的 Nginx 已经准备好开始使用了：

```sh
which nginx
# /usr/local/sbin/nginx

nginx -v
# nginx version: nginx/1.27.4
```

启动 Nginx

```sh
sudo nginx
curl localhost
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

成功了！

要删除 Nginx 包，只需要在 `/usr/local/stow` 目录下运行这条命令

```sh
sudo stow -D nginx-1.27.4
```

它会自动删除之前为 Nginx 包创建的软链接，这时我们就可以愉快地 `rm -rf` 了

```sh
sudo rm -rf nginx-1.27.4
```

这样我们就实现了对软件包的完美删除