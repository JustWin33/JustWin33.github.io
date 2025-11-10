---
title: Linux四剑客
description: 
date: 2025-11-10
categories:
    - 
    - 
---


# Linux四剑客

"Linux四剑客"是指在Linux系统中用于文件搜索和文本处理的四个核心命令行工具：**find、grep、sed、awk**。它们是系统管理和运维工作中不可或缺的利器，熟练掌握可大幅提升工作效率。

---

## 一、四剑客概述

| 命令     | 核心功能      | 处理对象   | 特点                           |
| -------- | ------------- | ---------- | ------------------------------ |
| **find** | 文件/目录查找 | 文件系统   | 递归搜索，支持多种条件组合     |
| **grep** | 文本内容搜索  | 文件内容   | 基于正则表达式匹配行           |
| **sed**  | 流式文本编辑  | 文本流     | 非交互式，擅长替换/删除/插入   |
| **awk**  | 文本分析处理  | 结构化文本 | 支持字段处理、计算和格式化输出 |

---

## 二、find：文件查找专家

### 2.1 基本语法
```bash
find [路径] [选项] [表达式]
```
- **路径**：搜索起始目录（默认为当前目录）
- **选项**：控制搜索行为
- **表达式**：定义匹配条件

### 2.2 核心操作步骤详解

**步骤1：按文件名查找**
```bash
find /etc/ -type f -name '*.conf'
find /etc/ -type f -name 'host*'      # 以hosts开头
find /bin/ /sbin/ -type f -name "*ip" # 文件名包含ip
```
- 操作意思：在指定目录下递归查找匹配名称的文件
- `-name`：按精确名称匹配（区分大小写），支持`*`（匹配一串字符）和`?`（匹配单个字符）
- `-iname`：忽略大小写匹配
- `-type f`：指定文件类型为普通文件（file）

**步骤2：按文件类型查找**
```bash
find /etc -type d      # 查找目录
find /etc -type l      # 查找软链接
```
- `-type`：指定文件类型，`d`（目录）、`f`（文件）、`l`（软链接）

**步骤3：按文件大小查找**
```bash
find /etc/ -type f -name "*.conf" -size +10k  # 大于10k
find / -size +100M     # 大于100MB的文件
find / -size -10k      # 小于10KB的文件
```
- `-size`：指定大小，`+1M`表示大于1M，`-1k`表示小于1k

**步骤4：按时间属性查找**
```bash
find /var/log/ -type f -name "*.log" -mtime +3  # 修改时间大于3天
find /var -mtime +7    # 7天前修改的文件
find /var -mtime -1    # 24小时内修改的文件
```
- `-mtime`：根据修改时间查找（单位：天），`+3`表示超过3天，`-1`表示1天内

**步骤5：根据深度和权限查找**
```bash
find /etc/ -maxdepth 1 -type f -name "*.conf"  # 最大深度1层
find / -perm 755        # 权限为755的文件
```
- `-maxdepth`：限制查找最大深度
- `-perm`：按权限查找

**步骤6：找出文件后进行删除（三种方法对比）**

创建测试环境：
```bash
mkdir -p /app/logs/
touch /app/logs/access{01..10}.log
```

**方法01：find与反引号**
```bash
rm -f `find /app/logs/ -type f -name '*.log'`
```
- 操作意思：先执行反引号中的find命令，将结果作为参数传递给rm删除
- **风险**：文件名含空格会导致错误

**方法02：find配合管道与xargs**
```bash
find /app/logs/ -type f -name "*.log" | xargs rm -f
```
- 操作意思：find查找结果通过管道传递给xargs，xargs将结果作为参数逐个执行rm命令
- **区别**：`|`传递的是字符串，`|xargs`传递的是命令参数

**方法03：find选项-exec**
```bash
find /app/logs/ -type f -name "*.log" -exec rm -f {} \;
```
- 操作意思：对find找到的每个文件执行rm命令
- `{}`：占位符，代表当前找到的文件
- `\;`：命令结束标记（必须转义）

**步骤7：find与-exec的高级用法**
```bash
find /etc/ -type f -name "*.conf" -exec tar zcf /backup/etc-exec.tar.gz {} +
```
- **坑**：使用`\;`会导致多次执行tar，最终只有1个文件
- **解决**：使用`+`号，先执行完find所有结果，一次性传递给exec
- 操作意思：`+` 让find将所有结果收集完成后，一次性传递给后面命令

**步骤8：找出文件后打包压缩**
```bash
# 方法01：find+反引号
tar zcf /backup/etc-conf.tar.gz `find /etc/ -type f -name "*.conf"`

# 方法02：find |xargs
find /etc/ -type f -name "*.conf" | xargs tar zcf /backup/etc_conf.tar.gz

# 方法03：find -exec（推荐+号）
find /etc/ -type f -name "*.conf" -exec tar zcf /backup/etc-exec.tar.gz {} +
```

---

## 三、grep：文本搜索利器

### 3.1 基本语法
```bash
grep [选项] "模式" [文件...]
```
- **模式**：支持基本正则表达式、扩展正则表达式或固定字符串
- **文件**：可指定多个文件，省略时从标准输入读取

### 3.2 核心操作步骤详解

**步骤1：基础文本搜索**
```bash
grep -n --color "root" /etc/passwd
```
- 操作意思：在文件中查找"root"，显示行号并高亮
- `-n`：显示行号
- `--color`：高亮匹配的关键字
- `/etc/passwd`：目标文件

**步骤2：行首行尾锚定**
```bash
grep -n --color "^root" /etc/passwd   # 以root开头
grep -n --color "bash$" /etc/passwd   # 以bash结尾
```
- `^`：字符串开头
- `$`：字符串结尾

**步骤3：反向匹配与去空行**
```bash
grep -v "#" /etc/passwd               # 不包含#的行
grep -v "#" /etc/passwd | grep -v "^$" # 不包含#且去除空行
grep -v "^$" file                     # 去除空行（^$表示空行）
```
- `-v`：反向选择，不包含匹配文本的行
- `"^$"`：匹配空行（开头即结尾）

**步骤4：忽略大小写搜索**
```bash
grep -i "May" test.txt                # 不区分大小写
```
- `-i`：ignore，忽略大小写

**步骤5：使用扩展正则（egrep）**
```bash
egrep '([0-9]{1,3}\.){3}[0-9]{1,3}$' test.txt  # 匹配IP地址
grep -E '([0-9]{1,3}\.){3}[0-9]{1,3}$' test.txt # 同上
```
- `egrep`：等同于`grep -E`，支持扩展正则表达式
- `{1,3}`：匹配1到3次
- `\.`：转义的点号

**步骤6：正则表达式高级用法**
```bash
# 匹配IP地址
grep --color "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}$" test.txt

# 简化版
egrep --color "([0-9]{1,3}\.){3}[0-9]{1,3}$" test.txt

# 其他示例
grep '[a-z]\{5\}' aa                 # 包含5个连续小写字母的行
grep -c "48" test.txt                # 统计以"48"开头的行数
```

**步骤7：多文件与管道过滤**
```bash
grep 'test' aa bb cc                 # 在多个文件中查找
ls -l | grep '^a'                    # 管道过滤，只显示以a开头的行
grep -l "root" /etc/*                # 只显示包含root的文件名
```

**步骤8：显示上下文**
```bash
grep -A 3 -B 2 "exception" app.log  # 显示匹配行的前后文
grep -C 5 "error" app.log            # 上下各显示5行
```

---

## 四、sed：流式文本编辑器

### 4.1 基本语法
```bash
sed [选项] '脚本命令' [文件]
```
- **脚本命令**：由地址和动作组成，如`'3d'`、`'s/old/new/g'`
- **默认行为**：处理结果输出到标准输出，**不修改原文件**

### 4.2 核心操作步骤详解

**步骤1：文本替换（最常用）**
```bash
sed 's/old/new/' file.txt            # 替换每行第一个"old"
sed 's/old/new/g' file.txt           # 替换每行所有"old"（全局）
sed 's/foo/bar/g' config.ini         # 全局替换
```
- `s`：substitute（替换）命令
- `/old/new/`：替换规则
- `g`：global，全局替换

**步骤2：直接修改文件（⚠️危险操作）**
```bash
sed -i 's/foo/bar/g' config.ini      # 直接修改文件（无备份）
sed -i.bak 's/foo/bar/g' config.ini  # 先备份为config.ini.bak再修改
```
- `-i`：in-place，直接修改原文件
- 建议：总是先测试再加`-i`，或使用备份

**步骤3：删除指定行**
```bash
sed '3d' file.txt                    # 删除第3行
sed '/^#/d' file.txt                 # 删除所有以#开头的行
sed '/^$/d' file.txt                 # 删除空行
```
- `d`：delete（删除）命令
- `/^#/`：正则匹配以#开头的行

**步骤4：插入/追加内容**
```bash
sed '3i\inserted line' file.txt      # 在第3行前插入
sed '3a\appended line' file.txt      # 在第3行后追加
```
- `i\`：insert（在指定行前插入）
- `a\`：append（在指定行后追加）

**步骤5：多命令组合**
```bash
sed -e 's/foo/bar/' -e 's/baz/qux/' file.txt
sed -e '3d' -e 's/old/new/g' file.txt
```
- `-e`：执行多个编辑命令（顺序执行）

---

## 五、awk：文本分析处理引擎

### 5.1 基本语法
```bash
awk '模式{动作}' [文件]
```
- **模式**：条件表达式或正则匹配
- **动作**：对匹配行执行的操作，多个动作用分号分隔
- **默认行为**：逐行处理，字段以空格或Tab分隔

### 5.2 核心操作步骤详解

**步骤1：提取指定列**
```bash
awk '{print $1}' access.log          # 提取第1列
awk '{print $1, $3, $NF}' file       # 提取第1、3列和最后一列
```
- `$1, $2...`：表示第1、2...个字段
- `$0`：整行内容
- `$NF`：最后一个字段（Number of Fields）

**步骤2：指定分隔符**
```bash
awk -F: '{print $1}' /etc/passwd     # 以冒号为分隔符，提取用户名
awk -F',' '{print $2}' data.csv      # 处理CSV文件
```
- `-F`：指定字段分隔符（支持正则，如`-F'[:;]'`）

**步骤3：条件过滤**
```bash
awk '$3 > 100' data.txt              # 输出第3列大于100的行
awk '/error/ {print $2}' app.log     # 匹配含"error"的行，输出第2列
```
- 模式可以是：正则表达式、比较表达式、组合条件

**步骤4：计算统计**
```bash
awk '{sum+=$1} END {print sum}' numbers.txt  # 求和
awk '{count++} END {print count}' file       # 统计行数
```
- `END`：在所有行处理完成后执行

**步骤5：格式化输出**
```bash
awk '{printf "%-10s %5d\n", $1, $2}' file
```
- `printf`：格式化输出（类似C语言）
- `%-10s`：左对齐，占10个字符的字符串
- `%5d`：右对齐，占5个字符的整数

**步骤6：数组与复杂统计**
```bash
# 统计每个IP的访问次数
awk '{ips[$1]++} END {for(ip in ips) print ip, ips[ip]}' access.log
```

---

## 六、特殊符号与引号详解

### 6.1 单引号、双引号、反引号、不加引号的区别

```bash
# echo 'oldboy $LANG hostname{01..10}'
# 输出：oldboy $LANG hostname{01..10}
# 单引号：所见即所得，内容原封不动输出，bash不会解析
```

```bash
# echo "oldboy $LANG hostname {01..10}"
# 输出：oldboy en_US.UTF-8 hostname {01..10}
# 双引号：解析$和``，不解析{}（通配符）
```

```bash
# echo oldboy $LANG hostname{01..10}
# 输出：oldboy en_US.UTF-8 hostname01 hostname02 ...
# 不加引号：与双引号类似，会解析{}（文件名通配符）
```

```bash
# echo `hostname`
# 输出：实际主机名
# 反引号：优先执行，先运行引号里面的命令（现代用法推荐$(command)）
```

### 6.2 重定向符号详解

| 符号            | 名称               | 作用                         |
| --------------- | ------------------ | ---------------------------- |
| `>` 或 `1>`     | 标准输出重定向     | 先清空文件，然后写入正确信息 |
| `>>` 或 `1>>`   | 标准输出追加重定向 | 追加到文件末尾               |
| `2>`            | 标准错误输出重定向 | 先清空，然后写入错误信息     |
| `2>>`           | 标准错误追加重定向 | 追加错误信息到文件末尾       |
| `2>&1` 或 `&>>` | 标准错误与输出合并 | 正确和错误信息都写入指定文件 |
| `<` 或 `0<`     | 输入重定向         | 从文件读取输入               |
| `<<` 或 `0<<`   | 追加输入重定向     | 从文件追加读取               |

**示例：**
```bash
command > file 2>&1    # 正确和错误都重定向到file
command &>> file       # 同上，简写形式
```

---

## 七、四剑客组合应用实例

### 实例1：清理过期日志并统计
```bash
find /var/log -name "*.log" -mtime +30 -exec gzip {} \; && \
find /var/log -name "*.gz" -mtime +365 -delete && \
grep -c "error" /var/log/app.log | awk '{sum+=$1} END {print "Total errors:", sum}'
```
**操作意思分解**：
1. 查找30天前的日志文件并压缩
2. 删除365天前的压缩包
3. 统计error出现总次数

### 实例2：配置批量修改与验证
```bash
sed -i.bak 's/timeout=30/timeout=60/g' /etc/app.conf && \
grep "timeout" /etc/app.conf | awk -F= '{print $2}' | sed 's/ //g'
```
**操作意思分解**：
1. 备份并修改超时配置
2. 提取新配置值并清理空格

### 实例3：日志分析报表（IP访问Top10）
```bash
grep "2024-01" access.log | \
awk '{ips[$1]++} END {for(ip in ips) print ip, ips[ip]}' | \
sort -k2 -nr | head -10
```
**操作意思分解**：
1. 筛选1月份日志
2. 统计每个IP的访问次数
3. 按访问次数排序，取前10名

---

## 八、注意事项与最佳实践

1. **find处理大量文件**：建议配合`-maxdepth`限制深度，使用`+`代替`\;`提升性能
2. **grep正则**：使用`-E`支持扩展正则，等价于`egrep`
3. **sed危险操作**：务必先测试再使用`-i`修改，防止误操作
4. **awk分隔符**：处理CSV文件时建议用`-F','`指定逗号分隔符
5. **引号使用**：优先使用单引号保护模式，需要变量解析时用双引号
6. **备份习惯**：任何修改操作前，先备份原始文件

掌握这四个工具的组合使用，可解决90%以上的Linux文本处理场景。建议通过`man find`、`man grep`、`man sed`、`man awk`查看完整手册深入学习。