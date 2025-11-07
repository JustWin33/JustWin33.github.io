---
title: Linux四剑客
description: 各自的用法
date: 2025-10-20
categories:
    - Test
    - 教程
---


Linux四剑客

1、find    ：           查找文件、目录，find与其他命令配合

2、gerp/egrep  ： 过滤

3、sed           ：过滤，替换，取行

4、awk           ：过滤，取行，取列，统计与计算



### find

find 路径 -name "文件名"

选项

`.`                        表示在当前目录下

`"ab*"`                    *匹配一串字符    ？匹配一个字符

`-type`                    d目录  f文件  l软链接

`-iname`                   忽略大小写查找含有关键字的文件

`-name`                    指定文件名，默认精确匹配。加上*模糊

`-size`                    指定大小 +1M表示大于1M的，-1k表示小于1k的文件

`-mtime`                   根据时间查找

`-maxdepth`                查找最大深度

`-perm 755`                表示权限为755的文件


#### 案例01：找出/etc/目录下面以.conf结尾的文件

`find /etc/ -type f -name '*.conf'`

find  目录  类型 名字

`find /etc/ -type f -name 'host*'`       #以hosts开头

找出/bin/  /sbin/  文件中包含ip的文件

`find /bin/ /sbin/ -type f -name "*ip"`

#### 案例02：找出/etc/目录下以.conf结尾的文件文件大小大于10k

`find /etc/ -type f -name "*.conf" -size +10k`

#### 案例03：找出/var/log下面以.log结尾的文件并且修改时间大于3天

`find /var/log/ -type f -name "*.log" -mtime +3`

#### 案例04：查找文件或目录的时候不区分大小写

`find /etc/ -type f -iname "*.conf"`

#### 案例根据深度查找文件

`find /etc/ -maxdepth 1 -type f -name "*.conf"`



#### 案例05：find找出文件后进行删除

创建测试环境

`mkdir  -p /app/logs/`

`touch /app/logs/access{01..10}.log`

方法01：find与反引导

rm -f `find /app/logs/ -type f -name '*.log'` 

方法02：find 管道

find /app/logs/ -type f -name "*.log" | xargs rm -f

| 与|xargs区别

|传递的是字符串，文字符号

|xargs传递是参数 命令后面文件，目录

方法03：find选项-exec

find -type f -name "*.log" -exec rm -f {} \;

-exec 命令 {} \;

{}前面find找出的文件内容

\;结尾标记



#### 案例06：找出/etc/下以.conf结尾的文件与打包压缩/backup/

方法01：find+反引号

tar zcf /backup/etc-conf.tar.gz  `find /etc/ -type f -name "*.conf"`

方法02：find |xargs

find /etc/ -type f -name "*.conf" |xargs tar zcf /backup/etc_conf.tar.gz

方法03：find -execf 

find /etc/ -type f -name "*.conf" -exec tar -zcf /backup/etc_conf.tar.gz {} \;

有坑，发现打包压缩后只有1个文件



find 与 -exe执行流程

find找出1个文件exec执行1次命令



find /etc/ -type f -name "*.conf" -exec tar zcf /backup/etc-exec.tar.gz {} +

+ 先执行前面命令执行完成，结果一次性通exec传递给后面命令







#### 特殊符号-重定向

##### 单引号、双引号、反引号、不加引号的区别

单引号：所见即所得，单引号里面的内容会原封不动输出，shell(bash)不会解析与运行

双引号：与单引号类似，特殊符号会被解析运行$,``,不会解析{}(通配符匹配文件名)

不加引号：与双引号类似，解析{}

反引号：优先执行，先运行引号里面的命令



#echo 'oldboy $LANG hostname{01..10}'| oldboy $LANG hostname {01..10}

#echo "oldboy $LANG hostname {01..10}"





##### 重定向符号 



`> 或1>`           标准输出重定向（正确信息），先清空，然后写入

`> > 或1>>`     标准输出追加重定向（正确信息），追加到文件末尾

`2>`                标准错误输出重定向（错误信息），先清空，然后写入

`2>>`              标准错误追加输出重定向（错误信息），追加到文件末尾

`> > oldboy.txt 2>&1 &>>oldboy.txt               标准错误信息都写入到指定文件`

`<或0<`            输入重定向

`<<或0<<`           追加输入重定向



2>&1:将正确和错误的信息都输出，适用于日志文件

2>:只输出错误的信息















### grep

{{< highlight html >}}
grep -n --color '^hello' ~/Desktop/2.txt
{{< /highlight >}}

 -n              ==> 显示行数

 -color          ==> 显示颜色

 '^hello'  ^     ==>字符串开头,查找的字符串
 $               ==> 字符串结尾

 ~/Desktop/2.txt ==> 查找的文件路径


{{< highlight html >}}
grep -v --color '^hello' ~/Desktop/2.txt | grep -v "^$"
{{< /highlight >}}
 -v        ==> 取反，不要以 hello 开头的内容

 -v ”^$”   ==> 去掉空行

{{< highlight html >}}
egrep '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$' 2.txt 
egrep '([0-9]{1,3}\.){3}[0-9]{1,3}$' 2.txt
{{< /highlight >}}
 '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$'  ==> 正则表达式匹配ip地址 xxx.xxx.xxx.xxx

cat /etc/passwd

grep 命令用于查找文件中的内容,匹配文件中的内容

grep  -n --color "root" /etc/passwd

 -n    ==>查找的结果在passwd文件的 行号

 --color  ==>为关键字加上颜色
 
 /etc/passwd  这个文件
 
 grep  -n --color "^root" /etc/passwd
 
 ^root 表示以root开头的行，文本后面出现root关键字没有用

{{< highlight html >}}
grep  -n --color "root$" /etc/passwd
grep  -n --color "bash$" /etc/passwd
{{< /highlight >}}
 root$ 表示文本所在行以root结尾

 bash$ 表示文本所在行以bash结尾

{{< highlight html >}}
grep "#" /etc/passwd  #查找passwd文件中包含#的行
grep -v "#" /etc/passwd  #查找passwd文件中不包含#的行
{{< /highlight >}}

grep -v "#" /etc/passwd | grep -v "^$"   #查找passwd文件中不包含#的行,并去除空行

grep -v "^$"   表示不包含空行，即去除空行

 -v  表示不包含
 
 "^$" 表示空行
 
 ^ 以空格开头
 
 $ 以空格结尾

grep -[a c i n v] '搜索的内容'  filename

 -a 以文本文件方式搜索

 -c 计算找到的符合行的次数

 -i 忽略大小写

 -n 顺便输出行号

 -v 反向选择，即显示不包含匹配文本的所有行

 -h 查询多文件时不显示文件名

 -l查询多文件时只输出包含匹配字符的文件名

 -s 不显示不存在或无匹配的错误信息

 grep 命令加 -E 参数，这一扩展允许使用扩展模式匹配

 grep -E  =  egrep

使用grep查找文件中的ip地址

{{< highlight html >}}
grep --color "[0-9][0-9]"  test.txt   
"[0-9]"  表示0~9任意一个字符
"[0-9][0-9]" 两个连续数字
grep --color "[0-9]\{1,3}"  test.txt  #匹配1~3次
{{< /highlight >}}

egrep --color "[0-9]\{1,3}\.[0-9]\{1,3}\.[0-9]\{1,3}\.[0-9]\{1,3}$"  test.txt
 
 [0-9]\{1,3}\. 表示以1~3位数字，并且后面有一个 . 号
 
 \.   表示转义的.  一定要加上
 
 [0-9]\{1,3}$   表示以1~3位数字结尾，如果是4位数字或以上就不行
 
 [a-z] 表示一个字母

egrep --color "([0-9]\{1,3}\.){3}[0-9]\{1,3}$"  test.txt 
{} 表示匹配的的次数
([0-9]\{1,3}\.){3} 将前面的形式匹配3次

[^] 匹配一个不在指定范围内的字符 ex: '[^A-FH-Z]rep' 匹配不包含A-R和T-Z的一个字母开头，紧跟rep的行。

ls -l |grep '^a'通过管道过滤ls -l 输出的内容，只显示以a开头的行
grep 'test' aa bb cc 显示在aa，bb，cc文件中匹配test行
grep '[a-z]/{5/}'  aa显示所有包含每个字符串至少有5个连续小写字符的字符串的行

grep -c "48" test.txt 统计所有以"48"字符开头的行有多少
grep -i "May" test.txt 不区分大小写查找 "May"所有的行

grep -n "May" test.txt 显示行号
grep -v "48" test.txt 显示输出没有字符"48" 所有的行
([a-z]+[a-z0-9])      表示匹配任意一个字母 + 任意一个数字或字母
([a-z]+[a-z0-9]+)    这边的+号表示可以匹配多个
















