---
title: "golang热加载及代码调试"
date: 2022-03-27T19:38:07+08:00
draft: false
---

## 热加载

### 为什么需要热加载

在编写go程序的过程中，每次修改完代码我们都得重新编译运行它，这是一个重复且麻烦的过程。那么有没有什么方法可以免去这个步骤，使得每次我们修改完代码自动编译并运行程序呢？

### 实现

这里要介绍一个用专门用于热加载golang程序的包 [air](https://github.com/cosmtrek/air)

#### 安装
```shell
$ go install github.com/cosmtrek/air@latest
```
#### 使用

1. 进入项目目录
2. 初始化配置文件
```shell
$ air init
```
3. 修改配置文件
实例文件
```toml
# Config file for [Air](https://github.com/cosmtrek/air) in TOML format

# 工作目录
# .或者绝对路径，注意下面的路径必须是相对于根路径的
root = "."
tmp_dir = "tmp"

[build]
  # 老的shell命令，也可以使用`make`
  cmd = "go build -o ./tmp/main ."
  # 从`cmd`产生的二进制文件.
  bin = "tmp/main"
  # 自定义编译二进制文件，可以在运行命令的时候设置环境变量
  full_bin = "APP_ENV=dev APP_USER=air ./tmp/main"
  # 监视扩展名为以下类型的文件
  include_ext = ["go", "tpl", "tmpl", "html"]
  # 忽略以下目录下的文件修改
  exclude_dir = ["assets", "tmp", "vendor", "frontend/node_modules"]
  # 监视这些目录下文件的修改
  include_dir = []
  # 忽略以下文件
  exclude_file = []
  # 排除符合以下正则表达式的文件
  exclude_regex = ["_test.go"]
  # 排除未修改的文件
  exclude_unchanged = true
  # 遵循目录的符号链接
  follow_symlink = true
  # 日志文件，将放在上面指定的临时目录下
  log = "air.log"
  # 修改文件后重新编译的延迟时间（当文件修改太过频繁的时候不需要每次一修改完立马出发编译）
  delay = 1000 # ms
  # 当编译出错后停止运行原来的二进制文件
  stop_on_error = true
  # 在杀死进程之前发送中断信号（windows不支持这个特性）
  send_interrupt = false
  # 发送中断信号后延迟多少面查杀进程
  kill_delay = 500 # ms

[log]
  # 是否显示日志时间
  time = false

[color]
  # 自定义每个部分的颜色。如果颜色未定义，则使用程序的原始日志
  main = "magenta"
  watcher = "cyan"
  build = "yellow"
  runner = "green"

[misc]
  # 是否在程序退出后删除临时目录
  clean_on_exit = true
```
4. 运行

使用默认路径`./.air.toml`
```shell
$ air
```
指定.air.toml路径
```shell
$ air -c ./.air.toml
```
向原程序传递参数
```shell
# 等同于 ./tmp/main bench
air bench

# 等同于 ./tmp/main server --port 8080
air server --port 8080
```

## 调试

这里以goland为例，在goland中，如果我们想要开启断点调试，只需要配置右上角的debug按钮即可，但是在上面所说的热加载中如何开启呢？

点击Run -> Edit Configurations点击面板上左上角的+号，选择Go Remote并保存
![img.png](/images/go_remote.png)

按照提示
1. 安装dlv
```shell
$ go install github.com/go-delve/delve/cmd/dlv@latest
```

2. 稍微修改一下上面air的full_bin配置

合并编译和运行两个步骤
```toml
full_bin="dlv debug --listen=:2345 --headless --api-version=2 --check-go-version=false --only-same-user=false --continue --accept-multiclient --output=./bin/server ./cmd/server/main.go"
```
3. 运行air
```shell
$ air
```

4. 选在刚才的配置，点击debug按钮

Enjoy!
