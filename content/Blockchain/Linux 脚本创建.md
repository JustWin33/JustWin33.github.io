---
title: Linux脚本创建
description: 
date: 2025-11-06
categories:
    - 
    - 
---
# Linux 脚本创建

## 一、脚本创建全流程详解

### 1.1 创建脚本的标准步骤

根据搜索结果，创建一个可执行的 Linux 脚本需要以下完整流程：

```bash
# 步骤1：创建文件
nano myscript.sh

# 步骤2：添加 Shebang 行（必须为第一行）
#!/bin/bash

# 步骤3：编写脚本代码
echo "Hello, World!"

# 步骤4：保存并退出
# (nano中按 Ctrl+O 保存，Ctrl+X 退出)

# 步骤5：添加执行权限
chmod +x myscript.sh

# 步骤6：执行脚本
./myscript.sh
```

---

## 二、文本编辑器完全指南

### 2.1 主流编辑器对比与详细操作

#### **Vim 编辑器（高级用户首选）**
```bash
# 创建并打开脚本文件
vim myscript.sh
```

**详细操作流程**：
1. 按 `i` 进入插入模式
2. 输入脚本内容
3. 按 `Esc` 退出插入模式
4. 输入 `:w` 保存文件
5. 输入 `:q` 退出vim
6. 或者组合使用 `:wq` 保存并退出
7. 如需强制退出不保存：`:q!`

**Vim 配置优化（智能脚本编写）**：
```bash
# 在 ~/.vimrc 文件中添加以下配置
syntax on           " 开启语法高亮
set number          " 显示行号
set tabstop=4       " 设置制表符宽度
set autoindent      " 自动缩进
set expandtab       " 将制表符转换为空格
```

**Vim 插件推荐**：
- **vim-bash-support**：提供 Bash 语法补全、代码片段
- **syntastic**：语法检查
- **NERDTree**：文件浏览器

---

#### **Nano 编辑器（新手友好）**
```bash
# 创建并打开脚本文件
nano myscript.sh
```

**操作快捷键**：
- `Ctrl+O`：保存文件（按 Enter 确认）
- `Ctrl+X`：退出编辑器
- `Ctrl+K`：剪切当前行
- `Ctrl+U`：粘贴
- `Alt+A`：标记文本块
- `Ctrl+W`：搜索文本
- `Alt+U`：撤销操作

**Nano 配置优化**：
```bash
# 编辑 /etc/nanorc 或 ~/.nanorc
set const            " 持续显示光标位置
set tabsize 4        " 设置制表符宽度
set autoindent       " 自动缩进
include /usr/share/nano/sh.nanorc  " 加载 Bash 语法高亮
```

---

#### **Gedit 图形界面编辑器**
```bash
# 在桌面环境下使用
gedit myscript.sh &
```

**特点**：
- 支持语法高亮、自动缩进
- 支持插件扩展
- 适合不熟悉命令行的用户

---

#### **VS Code 远程开发（现代推荐）**
```bash
# 安装 VS Code 和 Remote-SSH 插件
# 通过 SSH 直接编辑远程服务器脚本
code --remote=ssh-remote+user@host /path/to/script.sh
```

**优势**：
- 强大的 IntelliSense 自动补全
- 集成终端和调试器
- Git 版本控制集成

---

### 2.2 批量创建脚本模板

```bash
# 方法1：使用 cat 创建简单脚本
cat > myscript.sh << 'EOF'
#!/bin/bash
# Description: 脚本描述
# Author: 你的名字
# Date: $(date +%Y-%m-%d)

echo "这是一个快速创建的脚本"
EOF

# 方法2：使用 heredoc 创建多行脚本
cat > backup.sh << 'SCRIPT'
#!/bin/bash
# 自动备份脚本
set -euo pipefail

SOURCE_DIR="/data"
BACKUP_DIR="/backup/$(date +%Y%m%d_%H%M%S)"

echo "[$(date)] 开始备份..."
mkdir -p "$BACKUP_DIR"
cp -r "$SOURCE_DIR"/* "$BACKUP_DIR"/
echo "[$(date)] 备份完成: $BACKUP_DIR"
SCRIPT

# 方法3：使用 install 命令自动创建并设置权限
install -m 755 /dev/null myscript.sh
# 然后编辑文件
vim myscript.sh
```

---

## 三、脚本核心要素详解

### 3.1 Shebang 行深度解析

**Shebang 的作用**：
Shebang (`#!`) 告诉系统使用哪个解释器执行脚本，必须位于文件第一行。

**常见 Shebang 写法**：

| Shebang               | 解释器            | 适用场景                  |
| --------------------- | ----------------- | ------------------------- |
| `#!/bin/bash`         | Bash shell        | 大多数 Linux 系统（推荐） |
| `#!/usr/bin/env bash` | 环境变量查找 Bash | 跨平台兼容性好，优先使用  |
| `#!/bin/sh`           | POSIX shell       | 需要最大兼容性时使用      |
| `#!/bin/dash`         | Dash shell        | 追求脚本执行速度          |
| `#!/usr/bin/python3`  | Python 3          | Python 脚本               |
| `#!/usr/bin/perl`     | Perl              | Perl 脚本                 |
| `#!/bin/zsh`          | Zsh               | Zsh 特定功能              |

**最佳实践**：
```bash
#!/usr/bin/env bash
# 优点：
# 1. 使用 PATH 查找 bash，更灵活
# 2. 支持用户自定义 bash 版本
# 3. 在虚拟环境中表现更好
```

**Shebang 特殊用法**：
```bash
#!/bin/bash -e  # 执行时自动启用 errexit
#!/bin/bash -x  # 执行时自动启用调试模式
#!/bin/bash -eu # 同时启用 errexit 和 nounset
```

---

### 3.2 脚本标准结构模板

```bash
#!/usr/bin/env bash
################################################################################
# 脚本名称: myscript.sh
# 描述:     [脚本的详细功能描述]
# 作者:     [你的名字]
# 日期:     2024-12-30
# 版本:     1.0.0
# 依赖:     [列出所有依赖的命令和工具]
# 用法:     ./myscript.sh [选项] <参数>
# 示例:     ./myscript.sh -f config.conf
################################################################################

#===============================================================================
# 配置区 - 可修改的变量
#===============================================================================
SCRIPT_NAME="$(basename "$0")"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG_FILE="${SCRIPT_DIR}/config.conf"
LOG_FILE="/var/log/${SCRIPT_NAME%.*}.log"

#===============================================================================
# 安全设置 - 防御性编程
#===============================================================================
set -euo pipefail  # 遇到错误退出、变量未定义报错、管道失败检测
IFS=$'\n\t'        # 设置内部字段分隔符
umask 022          # 设置默认文件权限

# 错误处理函数
error_exit() {
    echo "[ERROR] $1" >&2
    exit 1
}

# 警告函数
warning() {
    echo "[WARNING] $1" >&2
}

# 信息输出函数
info() {
    echo "[INFO] $1"
}

#===============================================================================
# 通用函数定义区
#===============================================================================

# 检查命令是否存在
check_command() {
    command -v "$1" >/dev/null 2>&1 || error_exit "命令 '$1' 未找到"
}

# 检查是否为 root 用户
check_root() {
    if [[ $EUID -ne 0 ]]; then
        error_exit "此脚本必须以 root 权限运行"
    fi
}

# 打印帮助信息
print_help() {
    cat << EOF
用法: $SCRIPT_NAME [选项] <参数>

选项:
    -h, --help          显示此帮助信息
    -v, --version       显示版本信息
    -c, --config FILE   指定配置文件 (默认: $CONFIG_FILE)
    -d, --debug         启用调试模式

参数:
    <input_file>        输入文件路径

示例:
    $SCRIPT_NAME -d /path/to/file.txt

EOF
}

#===============================================================================
# 主逻辑区
#===============================================================================

# 解析命令行参数
while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            print_help
            exit 0
            ;;
        -v|--version)
            echo "$SCRIPT_NAME version 1.0.0"
            exit 0
            ;;
        -c|--config)
            CONFIG_FILE="$2"
            shift 2
            ;;
        -d|--debug)
            set -x  # 启用调试
            shift
            ;;
        *)
            POSITIONAL_ARGS+=("$1")
            shift
            ;;
    esac
done

# 主执行逻辑
main() {
    info "开始执行脚本..."
    
    # 检查依赖
    check_command "grep"
    check_command "awk"
    
    # 检查权限
    check_root
    
    # 读取配置文件
    if [[ -f "$CONFIG_FILE" ]]; then
        source "$CONFIG_FILE"
    else
        warning "配置文件不存在: $CONFIG_FILE"
    fi
    
    # 核心业务逻辑
    # ...
    
    info "脚本执行完成"
}

#===============================================================================
# 脚本入口
#===============================================================================
# 捕获错误并执行清理
trap 'error_exit "执行失败，行号: $LINENO"' ERR
trap 'info "脚本已终止"; exit 1' INT TERM

# 执行主函数
main "$@"
```

---

## 四、脚本编写核心要素

### 4.1 变量定义与使用

#### **变量命名规则**：
- 只能包含字母、数字和下划线
- 不能以数字开头
- 建议使用小写字母加下划线（如 `backup_dir`）
- 环境变量使用大写字母（如 `PATH`, `HOME`）

#### **变量定义方式**：

```bash
#!/usr/bin/env bash

# 基本变量
name="John"              # 字符串变量
age=20                   # 数字变量（bash 中均为字符串）
is_admin=true            # 布尔值（true/false 字符串）

# 命令替换（推荐 $() 语法）
current_date=$(date +%Y-%m-%d)
files_count=$(ls -1 | wc -l)

# 旧式反引号（不推荐，但兼容性好）
current_dir=`pwd`

# 变量引用
echo "名字: $name"
echo "年龄: $age"
echo "当前日期: $current_date"

# 大括号避免歧义
echo "今天是 ${current_date}_backup.tar.gz"

# 只读变量
readonly SCRIPT_VERSION="1.0.0"
# SCRIPT_VERSION="2.0"  # 会报错

# 删除变量
unset name
```

#### **特殊变量**：

| 变量            | 含义                   |
| --------------- | ---------------------- |
| `$0`            | 脚本名称               |
| `$1`, `$2`, ... | 位置参数               |
| `$#`            | 参数个数               |
| `$@`            | 所有参数（数组形式）   |
| `$*`            | 所有参数（字符串形式） |
| `$?`            | 上一个命令的退出状态   |
| `$$`            | 当前进程ID             |
| `$!`            | 最后一个后台进程ID     |
| `$-`            | 当前选项标志           |

---

### 4.2 参数处理高级技巧

```bash
#!/usr/bin/env bash

# 位置参数示例
echo "脚本名称: $0"
echo "第一个参数: $1"
echo "第二个参数: $2"
echo "所有参数: $@"
echo "参数个数: $#"

# 验证参数数量
if [[ $# -lt 2 ]]; then
    echo "用法: $0 <源目录> <目标目录>"
    exit 1
fi

# 处理可选参数
DEBUG=false
VERBOSE=false
while [[ $# -gt 0 ]]; do
    case $1 in
        -d|--debug)
            DEBUG=true
            set -x
            shift
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -*)
            echo "未知选项: $1"
            exit 1
            ;;
        *)
            POSITIONAL_ARGS+=("$1")
            shift
            ;;
    esac
done

# 重置位置参数
set -- "${POSITIONAL_ARGS[@]}"

# 使用 getopts（传统方式）
while getopts "hdv" opt; do
    case $opt in
        h)
            print_help
            exit 0
            ;;
        d)
            DEBUG=true
            ;;
        v)
            VERBOSE=true
            ;;
        \?)
            echo "无效选项: -$OPTARG" >&2
            exit 1
            ;;
    esac
done

# 使用 getopt（支持长选项，更强大）
OPTIONS=$(getopt -o hvd --long help,verbose,debug -- "$@")
if [[ $? -ne 0 ]]; then
    exit 1
fi
eval set -- "$OPTIONS"
```

---

### 4.3 条件判断与测试

#### **if 语句**：

```bash
#!/usr/bin/env bash

# 文件测试
if [[ -f "/etc/passwd" ]]; then
    echo "文件存在"
fi

# 目录测试
if [[ -d "/tmp" ]]; then
    echo "目录存在"
fi

# 字符串比较
name="admin"
if [[ "$name" == "admin" ]]; then
    echo "是管理员"
elif [[ "$name" == "user" ]]; then
    echo "是普通用户"
else
    echo "未知用户"
fi

# 数字比较
count=10
if [[ $count -eq 10 ]]; then
    echo "等于10"
elif [[ $count -gt 5 ]]; then
    echo "大于5"
fi

# 逻辑运算
age=25
if [[ $age -ge 18 && $age -le 60 ]]; then
    echo "工作年龄"
fi

# 模式匹配
filename="script.sh"
if [[ $filename == *.sh ]]; then
    echo "是 shell 脚本"
fi

# 正则表达式匹配
if [[ $email =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "有效的邮箱格式"
fi
```

#### **case 语句**：

```bash
#!/usr/bin/env bash

read -p "输入选择 (start/stop/restart): " action

case "$action" in
    start)
        echo "启动服务..."
        systemctl start nginx
        ;;
    stop)
        echo "停止服务..."
        systemctl stop nginx
        ;;
    restart)
        echo "重启服务..."
        systemctl restart nginx
        ;;
    *)
        echo "无效选项: $action"
        exit 1
        ;;
esac
```

---

### 4.4 循环结构

#### **for 循环**：

```bash
#!/usr/bin/env bash

# 遍历数字
for i in {1..5}; do
    echo "数字: $i"
done

# 遍历文件
for file in *.sh; do
    echo "处理文件: $file"
    chmod +x "$file"
done

# C 风格 for 循环
for ((i=0; i<10; i++)); do
    echo "计数: $i"
done

# 遍历数组
apps=("nginx" "mysql" "redis")
for app in "${apps[@]}"; do
    echo "检查服务: $app"
    systemctl status "$app"
done

# 读取文件内容
while IFS= read -r line; do
    echo "行内容: $line"
done < /etc/passwd

# 遍历命令输出
for user in $(awk -F: '{print $1}' /etc/passwd); do
    echo "用户: $user"
done
```

#### **while 循环**：

```bash
#!/usr/bin/env bash

# 条件循环
count=1
while [[ $count -le 5 ]]; do
    echo "计数: $count"
    ((count++))
done

# 无限循环（带退出条件）
while true; do
    read -p "输入 'exit' 退出: " input
    if [[ "$input" == "exit" ]]; then
        break
    fi
    echo "你输入了: $input"
done

# 读取文件
while IFS=: read -r username password uid gid info home shell; do
    echo "用户: $username, UID: $uid, HOME: $home"
done < /etc/passwd

# 带条件的 while
while [[ -f "/tmp/lockfile" ]]; do
    echo "等待锁文件删除..."
    sleep 5
done
```

---

### 4.5 函数定义与使用

```bash
#!/usr/bin/env bash

# 基本函数定义
function say_hello() {
    echo "Hello, $1!"
}

# 或者不使用 function 关键字
say_goodbye() {
    echo "Goodbye, $1!"
}

# 调用函数
say_hello "World"
say_goodbye "World"

# 带返回值的函数
check_disk_usage() {
    local threshold=80
    local usage=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
    
    if [[ $usage -gt $threshold ]]; then
        echo "警告: 磁盘使用率超过 ${threshold}%"
        return 1
    else
        echo "磁盘使用率正常: ${usage}%"
        return 0
    fi
}

# 捕获返回值
if check_disk_usage; then
    echo "检查通过"
else
    echo "检查失败"
fi

# 返回数据
get_config_value() {
    local key="$1"
    grep "^${key}=" config.conf | cut -d= -f2
}

# 使用返回值
db_host=$(get_config_value "DB_HOST")
echo "数据库主机: $db_host"

# 递归函数
factorial() {
    local n="$1"
    if [[ $n -le 1 ]]; then
        echo 1
    else
        echo $(( n * $(factorial $((n-1))) ))
    fi
}

result=$(factorial 5)
echo "5! = $result"
```

---

## 五、高级技巧与最佳实践

### 5.1 错误处理机制

```bash
#!/usr/bin/env bash

# 启用严格模式（推荐放在脚本开头）
set -euo pipefail

# 详细解释：
# set -e: 遇到错误立即退出（errexit）
# set -u: 使用未定义变量时报错（nounset）
# set -o pipefail: 管道命令中任何失败都认为是失败

# 自定义错误处理
trap 'error_handler $? $LINENO' ERR

error_handler() {
    local exit_code=$1
    local line_no=$2
    echo "错误发生在第 $line_no 行，退出码: $exit_code"
    cleanup
    exit $exit_code
}

# 清理函数
cleanup() {
    echo "清理临时文件..."
    rm -f /tmp/temp_file_$$
}

# 确保 cleanup 在脚本退出时执行
trap cleanup EXIT

# 检查命令是否成功
if ! command arg1 arg2; then
    error_handler $? $LINENO
fi

# 或者使用 ||
command arg1 arg2 || {
    echo "命令失败"
    exit 1
}

# 使用 $?
mkdir /tmp/mydir
if [[ $? -ne 0 ]]; then
    echo "创建目录失败"
    exit 1
fi
```

---

### 5.2 调试技巧大全

```bash
#!/usr/bin/env bash

# 方法1：在 Shebang 中启用调试
#!/bin/bash -x

# 方法2：在脚本中使用 set 命令
set -x  # 开启调试
# ... 需要调试的代码 ...
set +x  # 关闭调试

# 方法3：调试特定代码块
{
    set -x
    # 调试这段代码
    ls -l /nonexistent_dir
    set +x
}

# 方法4：打印变量值
debug() {
    if [[ "${DEBUG:-false}" == "true" ]]; then
        echo "[DEBUG] $*"
    fi
}

DEBUG=true
debug "变量值: name=$name"

# 方法5：使用 BASH_XTRACEFD 重定向调试输出
exec 5> debug.log
BASH_XTRACEFD=5
set -x

# 方法6：显示完整的命令执行
set -v  # 显示原始命令
set -x  # 显示扩展后的命令

# 方法7：使用 PS4 自定义调试前缀
export PS4='+(${BASH_SOURCE}:${LINENO}) ${FUNCNAME:+$FUNCNAME(): }'
set -x
```

---

### 5.3 代码规范与风格

**Google Shell Style Guide 推荐规范**：

```bash
#!/usr/bin/env bash

# 1. 变量命名：小写加下划线
local backup_dir="/backup"

# 2. 常量：大写加下划线
readonly BACKUP_RETENTION_DAYS=30

# 3. 函数命名：小写加下划线，使用动词开头
check_permissions() { ... }
backup_files() { ... }

# 4. 缩进：使用2个空格（不是制表符）
function main() {
  if [[ ... ]]; then
    echo "缩进2个空格"
  fi
}

# 5. 行长：不超过80字符
if [[ "$very_long_condition" == "something" && \
      "$another_condition" == "something_else" ]]; then
  echo "换行"
fi

# 6. 引用变量：总是使用双引号
echo "$variable"  # 正确
echo $variable    # 错误

# 7. 命令替换：使用 $() 而不是反引号
current_dir=$(pwd)  # 正确
current_dir=`pwd`   # 错误

# 8. 条件判断：使用 [[ ]] 而不是 [ ]
if [[ $var == "value" ]]; then  # 正确
if [ $var = "value" ]; then     # 错误（易出错）

# 9. 注释规范
# 单行注释

# 多行注释
# 可以跨越多行
# 说明复杂逻辑

: '
这是另一种多行注释方式
使用 : 和单引号
'

# 10. 函数文档
#######################################
# 检查磁盘空间
# Arguments:
#   阈值百分比
# Returns:
#   0 如果空间充足, 1 如果空间不足
#######################################
check_disk_space() { ... }
```

---

## 六、实用开发工具链

### 6.1 ShellCheck 代码检查

```bash
# 安装 ShellCheck
yum install -y epel-release
yum install -y ShellCheck

# 使用方式
shellcheck myscript.sh

# 在 Vim 中集成
# 在 ~/.vimrc 中添加
autocmd FileType sh setlocal makeprg=shellcheck\ -f\ gcc\ %
autocmd FileType sh setlocal errorformat=%f:%l:%c:\ %m

# 在 VS Code 中安装 ShellCheck 插件
```

---

### 6.2 Bash 自动补全

```bash
# 为脚本添加自动补全
_complete_myscript() {
    local cur prev opts
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    opts="-h --help -v --version -d --debug"

    if [[ ${cur} == -* ]]; then
        COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
        return 0
    fi
}

complete -F _complete_myscript myscript.sh
```

---

### 6.3 日志记录框架

```bash
#!/usr/bin/env bash

# 日志级别
LOG_LEVEL_DEBUG=0
LOG_LEVEL_INFO=1
LOG_LEVEL_WARN=2
LOG_LEVEL_ERROR=3

# 当前日志级别
CURRENT_LOG_LEVEL=${LOG_LEVEL_INFO}

# 日志函数
log() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local level_str

    case $level in
        $LOG_LEVEL_DEBUG) level_str="DEBUG" ;;
        $LOG_LEVEL_INFO)  level_str="INFO"  ;;
        $LOG_LEVEL_WARN)  level_str="WARN"  ;;
        $LOG_LEVEL_ERROR) level_str="ERROR" ;;
    esac

    if [[ $level -ge $CURRENT_LOG_LEVEL ]]; then
        echo "[$timestamp] [$level_str] $message" | tee -a "$LOG_FILE"
    fi
}

debug() { log $LOG_LEVEL_DEBUG "$@"; }
info()  { log $LOG_LEVEL_INFO  "$@"; }
warn()  { log $LOG_LEVEL_WARN  "$@"; }
error() { log $LOG_LEVEL_ERROR "$@"; }

# 使用示例
debug "调试信息"
info "普通信息"
warn "警告信息"
error "错误信息"
```

---

## 七、实战案例库

### 7.1 系统巡检脚本

```bash
#!/usr/bin/env bash
################################################################################
# 系统健康检查脚本
################################################################################

set -euo pipefail

# 检查CPU使用率
check_cpu() {
    local usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    if (( $(echo "$usage > 80" | bc -l) )); then
        echo "WARNING: CPU使用率过高: ${usage}%"
    else
        echo "OK: CPU使用率正常: ${usage}%"
    fi
}

# 检查内存使用率
check_memory() {
    local total=$(free -m | awk 'NR==2{print $2}')
    local used=$(free -m | awk 'NR==2{print $3}')
    local usage=$(echo "scale=2; $used*100/$total" | bc)
    
    if (( $(echo "$usage > 85" | bc -l) )); then
        echo "WARNING: 内存使用率过高: ${usage}%"
    else
        echo "OK: 内存使用率正常: ${usage}%"
    fi
}

# 检查磁盘使用率
check_disk() {
    df -h | awk 'NR>1 && $5+0 > 80 {print "WARNING: 磁盘使用率过高: " $0}'
}

# 主函数
main() {
    echo "===== 系统巡检报告 ($(date)) ====="
    check_cpu
    check_memory
    check_disk
    echo "===== 巡检完成 ====="
}

main "$@"
```

---

### 7.2 自动化备份脚本（带版本控制）

```bash
#!/usr/bin/env bash
################################################################################
# 自动化备份脚本 - 保留7天历史
################################################################################

set -euo pipefail

# 配置
SOURCE_DIR="/data/website"
BACKUP_BASE="/backup"
RETENTION_DAYS=7
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="website_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_BASE}/${BACKUP_NAME}"

# 创建备份
create_backup() {
    echo "[$(date)] 开始创建备份..."
    
    # 创建备份目录
    mkdir -p "$BACKUP_BASE"
    
    # 执行备份（排除日志和临时文件）
    tar -czf "$BACKUP_PATH" \
        --exclude="*.log" \
        --exclude="temp/*" \
        -C "$(dirname "$SOURCE_DIR")" \
        "$(basename "$SOURCE_DIR")"
    
    # 验证备份
    if [[ -f "$BACKUP_PATH" ]]; then
        echo "[$(date)] 备份成功: $BACKUP_PATH"
        ls -lh "$BACKUP_PATH"
    else
        echo "[$(date)] 备份失败!" >&2
        exit 1
    fi
}

# 清理旧备份
cleanup_old() {
    echo "[$(date)] 清理 ${RETENTION_DAYS} 天前的旧备份..."
    find "$BACKUP_BASE" -name "website_*.tar.gz" -mtime +${RETENTION_DAYS} -delete
    echo "[$(date)] 清理完成"
}

# 主流程
main() {
    create_backup
    cleanup_old
}

main "$@"
```

---

### 7.3 Nginx 日志分析脚本

```bash
#!/usr/bin/env bash
################################################################################
# Nginx 访问日志分析脚本
################################################################################

set -euo pipefail

LOG_FILE="/var/log/nginx/access.log"
REPORT_FILE="/tmp/nginx_report_$(date +%Y%m%d).txt"

# 检查日志文件
if [[ ! -f "$LOG_FILE" ]]; then
    echo "错误: 日志文件不存在: $LOG_FILE" >&2
    exit 1
fi

# 分析IP地址
analyze_ips() {
    echo "===== Top 10 IP 地址 =====" > "$REPORT_FILE"
    awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -nr | head -10 >> "$REPORT_FILE"
}

# 分析状态码
analyze_status() {
    echo -e "\n===== 状态码统计 =====" >> "$REPORT_FILE"
    awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -nr >> "$REPORT_FILE"
}

# 分析访问最多的URL
analyze_urls() {
    echo -e "\n===== Top 10 URL =====" >> "$REPORT_FILE"
    awk '{print $7}' "$LOG_FILE" | sort | uniq -c | sort -nr | head -10 >> "$REPORT_FILE"
}

# 生成报告
generate_report() {
    analyze_ips
    analyze_status
    analyze_urls
    
    echo "报告已生成: $REPORT_FILE"
    cat "$REPORT_FILE"
}

# 邮件发送
send_email() {
    local recipient="admin@example.com"
    mail -s "Nginx 日志报告" "$recipient" < "$REPORT_FILE"
}

main() {
    generate_report
    send_email
}

main "$@"
```

---

### 7.4 安全初始化脚本（CentOS）

```bash
#!/usr/bin/env bash
################################################################################
# CentOS 系统安全初始化脚本
################################################################################

set -euo pipefail

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
    exit 1
}

# 检查是否为 root
[[ $EUID -eq 0 ]] || error "此脚本必须以 root 身份运行"

# 更新系统
update_system() {
    log "更新系统..."
    yum update -y
}

# 配置防火墙
configure_firewall() {
    log "配置防火墙..."
    systemctl enable firewalld
    systemctl start firewalld
    
    # 开放 SSH 端口
    firewall-cmd --permanent --add-service=ssh
    
    # 开放 HTTP/HTTPS
    firewall-cmd --permanent --add-service=http
    firewall-cmd --permanent --add-service=https
    
    firewall-cmd --reload
}

# 配置 SSH
configure_ssh() {
    log "加固 SSH 配置..."
    
    # 备份原始配置
    cp /etc/ssh/sshd_config{,.bak.$(date +%Y%m%d)}
    
    # 修改配置
    sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
    
    # 重启 SSH
    systemctl restart sshd
}

# 安装 fail2ban
install_fail2ban() {
    log "安装 fail2ban..."
    yum install -y epel-release
    yum install -y fail2ban
    
    # 配置
    cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 3600
EOF
    
    systemctl enable fail2ban
    systemctl start fail2ban
}

# 主流程
main() {
    log "开始系统安全初始化..."
    
    update_system
    configure_firewall
    configure_ssh
    install_fail2ban
    
    log "安全初始化完成！"
    log "请重新登录以应用 SSH 更改"
}

main "$@"
```

---

## 八、开发工作流建议

### 8.1 版本控制

```bash
# 初始化 Git 仓库
git init
git add myscript.sh
git commit -m "Initial commit"

# 添加版本信息到脚本
VERSION="1.0.0"
git tag -a "v$VERSION" -m "Release version $VERSION"

# 自动生成版本
VERSION=$(git describe --tags --always --dirty)
sed -i "s/^VERSION=.*/VERSION=\"$VERSION\"/" myscript.sh
```

### 8.2 测试框架

```bash
#!/usr/bin/env bash
# test_myscript.sh

# 引入测试框架（如 Bats）
load 'libs/bats-support/load'
load 'libs/bats-assert/load'

# 测试函数
@test "检查函数返回值" {
    result=$(my_function "test")
    [[ "$result" == "expected" ]]
}

@test "检查文件创建" {
    run create_file "/tmp/testfile"
    [[ -f "/tmp/testfile" ]]
}
```

### 8.3 持续集成

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
      - name: Test script
        run: |
          chmod +x myscript.sh
          ./myscript.sh test
```

---

## 九、总结与资源

### 9.1 学习路径建议

1. **入门**（1-2周）：掌握基础语法、变量、条件、循环
2. **进阶**（2-4周）：学习函数、错误处理、调试技巧
3. **高级**（1-2月）：掌握性能优化、安全编程、复杂项目管理
4. **专家**：贡献开源项目、编写可复用库

### 9.2 必备资源

- **在线教程**：
  - [Bash Academy](https://www.bash.academy/)
  - [ShellCheck 规则文档](https://github.com/koalaman/shellcheck/wiki)
  
- **书籍**：
  - 《Linux命令行与Shell脚本编程大全》
  - 《Bash Cookbook》
  
- **社区**：
  - Stack Overflow: [bash] 标签
  - Unix & Linux Stack Exchange

### 9.3 快速参考卡

```bash
# 创建并执行脚本的最简流程
cat > script.sh << 'EOF'
#!/usr/bin/env bash
# Script description
set -euo pipefail

main() {
    echo "Hello, World!"
}

main "$@"
EOF

chmod +x script.sh
./script.sh
```

---

**最终建议**：对于你的 `install_ops_tools_centos.sh` 脚本，建议按照以下结构重构：

```bash
#!/usr/bin/env bash
################################################################################
# 运维工具安装脚本
# Description: 自动化安装常用运维工具
# Author: 你的名字
# Date: 2024-12-30
################################################################################

set -euo pipefail

# 颜色定义
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
    exit 1
}

check_root() {
    [[ $EUID -eq 0 ]] || error "需要 root 权限"
}

install_neofetch() {
    log "安装 neofetch..."
    yum install -y neofetch || error "neofetch 安装失败"
}

main() {
    check_root
    install_neofetch
    log "安装完成！请手动运行 vim +PlugInstall 和 tmux 插件安装"
}

main "$@"
```

这样你的脚本将更加专业、健壮和易于维护。