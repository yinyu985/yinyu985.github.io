---
title: Shell 学习之 getopt 和 getopts
slug: shell-learning-getopt-and-getopts
tags: [Linux, Shell]
date: 2022-09-11T00:25:01+08:00
---

`getopt` 与 `getopts` 都是 Bash 中用来获取与分析命令行参数的工具，常用在 Shell 脚本中被用来分析脚本参数。

- `getopts` 是 Shell 内建命令，`getopt` 是一个独立外部工具
- `getopts` 使用语法简单，`getopt` 使用语法较复杂
- `getopts` 不支持长参数（如：`--option`），`getopt` 支持长参数
- `getopts` 出现的目的是为了在不太复杂的场景代替 `getopt` 较快捷地执行参数分析工作
- `getopts` 负责参数解析，可以方便地提取参数值，`getopt` 只负责按规则重新对参数进行排列，进一步解析需要自行编写代码处理

<!--more-->

> 当脚本需要传入多个变量时，使用普通的 `$1`, `$2` 来传递变量，就显得有点笨了（需要记住变量顺序，还要记得默认的可选参数）

我们更希望通过类似下方式来传参：

```bash
myscript -u username -p password -v -n 9999 192.168.1.2
```

这时 `getopt` / `getopts` 就可以大展拳脚了。

根据前面提到的案例：

```bash
myscript -u username -p password -v -n 9999 192.168.1.2
```

| 参数       | 说明     |
| ---------- | -------- |
| `-u`       | 用户名   |
| `-p`       | 密码     |
| `-n`       | 端口     |
| `-v`       | 显示详情 |
| 无名称参数 | 主机 IP  |

```bash
#!/bin/bash
# 处理脚本参数
# -u 用户名
# -p 密码
# -v 是否显示详情
# -n 端口
while getopts ":u:p:n:v" opt_name
do
    case "$opt_name" in
        'u')
            CONN_USERNAME="$OPTARG"
            ;;
        'p')
            CONN_PASSWORD="$OPTARG"
            ;;
        'v')
            CONN_SHOW_DETAIL=true
            ;;
        'n')
            CONN_PORT="$OPTARG"
            ;;
        ?)
            echo "Unknown argument(s)."
            exit 2
            ;;
    esac
done

# 删除已解析的参数
shift $((OPTIND-1))
# 也可以将 shift 写进每个 option 里面，shift 命令用于对参数的移动（左移），通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理（常见于 Linux 中各种程序的启动脚本）。
# shift (shift 1) 命令每执行一次，变量的个数 ($#) 减一（之前的 $1 变量被销毁，之后的 $2 就变成了 $1），而变量值提前一位。同理，shift n 后，前 n 位参数都会被销毁

# 通过第一个无名称参数获取主机
CONN_HOST="$1"

# 显示获取参数结果
echo 用户名      "$CONN_USERNAME"
echo 密码        "$CONN_PASSWORD"
echo 主机        "$CONN_HOST"
echo 端口        "$CONN_PORT"
echo 显示详情     "$CONN_SHOW_DETAIL"
```

`:u:p:n:v` 就是指定要解析的参数名称。

### 规则说明

- 其中的字母表示需要解析的参数名称
- 字母后面的冒号 `:` 表示该参数除了其本身，还会带上一个参数作为选项的值，传入的值通过 `$OPTARG` 变量获取
- 字母后面**没有**冒号 `:` 表示该参数为开关型选项，不需要再指定值，只作为是否存在的标记
- 字符串开头的冒号 `:` 表示解析过程中，遇到未在 `getopts` 参数列表中指定的参数，不显示报错信息。否则会报出错误

使用 `getopts` 解析参数时，按照指定参数列表依次进行解析。如果本次解析符合指定参数规则，包括参数名称、是否需要传值等规则，则返回成功，进行下一次循环继续解析，否则退出循环。

### 失败规则

- 遇到未定义的变量
- 遇到了意外的值，如：在不需要传值的参数后面指定了参数，或者传入了比期待更多的值

失败后退出循环。

**注意！！！不带名称的参数一定要写到最后！否则会被认为是不期待的参数，导致停止解析。**

### `getopts` 的局限

对于常用的不太复杂的场景，使用 `getopts` 处理参数基本够用，也更方便，而且是内部命令，不用考虑安装问题，但也有一些局限：

- 选项参数的格式必须是 `-d val` 而不能是中间没有空格的 `-dval`
- 所有`选项参数`必须写在其它参数的前面，因为 `getopts` 是从命令行前面开始处理，遇到非 `-` 开头的参数，或者选项参数结束标记 `--` 就中止了，如果中间遇到`非选项命令行参数`，后面的选项参数就都取不到了
- 不支持 `--debug` 这样的长选项

### `getopt` 使用说明

`getopt` 是 [util-linux](https://en.wikipedia.org/wiki/Util-linux) 包中的一个命令，Linux 中基本都已预安装了 `getopt`，样例脚本一般安装到如下位置：

```bash
/usr/share/doc/util-linux-2.23.2
/usr/share/getopt/
/usr/share/docs/
```

本样例参考了如下脚本：

```bash
/usr/share/doc/util-linux-2.23.2/getopt-parse.bash
```

macOS 自带的 `getopt` 功能比较弱，不支持长选项，可以安装 GNU 版本 `gnu-getopt`：

```bash
brew install gnu-getopt
```

查看 getopt 的帮助信息：

```bash
$ getopt --help

用法：
 getopt optstring parameters
 getopt [options] [--] optstring parameters
 getopt [options] -o|--options optstring [options] [--] parameters

选项：
 -a, --alternative            允许长选项以 - 开始
 -h, --help                   这个简短的用法指南
 -l, --longoptions <长选项>    要识别的长选项
 -n, --name <程序名>           将错误报告给的程序名
 -o, --options <选项字符串>     要识别的短选项
 -q, --quiet                  禁止 getopt(3) 的错误报告
 -Q, --quiet-output           无正常输出
 -s, --shell <shell>          设置 shell 引用规则
 -T, --test                   测试 getopt(1) 版本
 -u, --unquoted               不引用输出
 -V, --version                输出版本信息
```

根据前面提到的案例，这里增加一个`日志级别`选项，此选项有默认值，也可以自行指定参数值。

```bash
# 短参数格式
$ myscript -u username -p password -v -n 9999 192.168.1.2 -l3
# 或长参数格式
$ myscript --username username --password password --verbose --port 9999 192.168.1.2 --log-level=3
```

### 参数说明

| 参数                | 说明                   |
| ------------------- | ---------------------- |
| `-u`, `--username`  | 用户名                 |
| `-p`, `--password`  | 密码                   |
| `-n`, `--port`      | 端口                   |
| `-v`, `--verbose`   | 显示详情               |
| `-l`, `--log-level` | 日志级别，默认级别为 1 |
| 无名称参数          | 主机                   |

```bash
#!/bin/bash

# 使用 "$@" 来让每个命令行参数扩展为一个单独的单词。"$@" 周围的引号是必不可少的！
ARGS=$(getopt -o 'u:p:n:vl::' -l 'username:,password:,port:,verbose,log-level::' -- "$@")

if [ $? != 0 ]; then
    echo "Parse error! Terminating..." >&2
    exit 1
fi

# 将参数设置为 getopt 整理后的参数
# $ARGS 需要用引号包围
eval set -- "$ARGS"

# 循环解析参数
while true; do
    case "$1" in
        -u|--username)
            CONN_USERNAME="$2"
            shift 2
            ;;
        -p|--password)
            CONN_PASSWORD="$2"
            shift 2
            ;;
        -n|--port)
            CONN_PORT="$2"
            shift 2
            ;;
        -v|--verbose)
            CONN_SHOW_DETAIL=true
            shift
            ;;
        -l|--log-level)
            case "$2" in
                "")
                    CONN_LOG_LEVEL=1
                    shift 2
                    ;;
                *)
                    CONN_LOG_LEVEL="$2"
                    shift 2
                    ;;
            esac
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Internal error!"
            exit 1
            ;;
    esac
done

# 通过第一个无名称参数获取主机
CONN_HOST="$1"

# 显示获取参数结果
echo '用户名：    '  "$CONN_USERNAME"
echo '密码：      '  "$CONN_PASSWORD"
echo '主机：      '  "$CONN_HOST"
echo '端口：      '  "$CONN_PORT"
echo '显示详情：  '  "$CONN_SHOW_DETAIL"
echo '日志级别：  '  "$CONN_LOG_LEVEL"
```

### 总结

其实 `getopt` 只负责做参数的重新整理，并不管提取参数值。它会根据指定的参数列表，把命令行中的选项参数集中放到前面，仅此而已。这样处理之后，再自己通过代码进行解析就比较简单了。所以上面的代码样例，真正涉及 getopt 使用的只有一行，其余的代码都是配合 getopt 重新排列的参数，自行进一步解析而已。

在本例中，`选项参数`与`非选项参数`没有按顺序排列，所以先告诉 `getopt` 命令要解析哪些参数：

```bash
getopt -o 'u:p:n:vl::' -l 'username:,password:,port:,verbose,log-level::' -- "$@"
```

### 参数规则

- `-o` 参数指定短参数格式，`-l` 参数指定对应的长参数
- 冒号 `:` 规则与 `getopts` 的规则基本一致。区别在于后面带有两个冒号 `::` 的表示默认值参数
- 对于`默认值选项`，短参数形式参数名与值之间不能有空格，长参数形式参数名与值需要用等号 `=` 连接