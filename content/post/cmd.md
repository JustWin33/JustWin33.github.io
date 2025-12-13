---
title: CMD-1
description: 示例 
date: 2025-12-13
categories:
    - 
    - 
---
cmd





```bash
#1、简单查询当前用户
whoami 
2、查询详细权限组
whoami /groups

3、查询当前用户的详细信息
whoami /user


4、检查查看IP地址
ipconfig
5、查看详细信息（包含MAC地址、DNS服务器）
ipconfig /all

6、刷新DNS缓存
ipconfig /flushdns

7、显示所有系统信息
systeminfo

8、管道过滤（只看内存信息）
systeminfo | findstr "Memory"

9、查看简短的系统版本号
ver

10、列出所有正在运行的进程
tasklist

11、查找特定的进程
tasklist | findster "notepad"

12、强制结束进程
taskkill /F /IM notepad.exe


```



```bash
文件系统 User Profile:
你的位置在：C:\Users\win16(也就是%USERPROFILE%),任何Windows都有：
1、Desktop(桌面)：展示台，放你最常用的东西
2、Downloads(下载)：仓库，浏览器下载的东西默认丢这
3、Documents(文档)：书房，存你的Word/Excel文件
4、Pictures(图片)
5、Music（音乐）&Videos(视频)
```



```bash
1、回到用户主目录
cd %USERPROFILE%
2、以树状图显示当前目录结构
/F=显示文件，不加只显示文件夹
tree /F


md(Make Directory):建房
echo:喊话，配合>(重定向符)可以把话“写”进文件里

创建一个练习用的文件夹
md MyLab
进入
cd MyLab
创建一个文件并写入内容
echo Hello Windows > demo.txt
意思是：把“Hello Windows"这句话，写入demo.txt
确认文件是否存在
dir



读取内容
type:打印文件内容到屏幕
查看demo.txt的内容
type demo.txt

销毁
del:删除文件
rd:删除文件夹
删除刚才的文件
del demo.txt
退回上一级目录
cd ..
删除MyLab文件夹
rd MyLab
默认只能删空文件夹，如果里面有东西，需要加 /S /Q



例1：在桌面上创建一个叫LogTest的文件夹，进入该文件夹，用echo创建一个叫status.txt的文件，内容写system all green.用type读取它

cd /d %USERPROFILE%
cd Desktop
md MyDownloadTest


例3：去文档目录写一个日记
cd /d %USERPROFILE%\Documents
echo 2025-10-16 Study Windows > diary.txt
type diary.txt

有时候在黑黑的窗口里操作，确认自己在哪里，有一个好用的命令：start .
start :启动打开
.（点）:代表当前目录

在CMD里输入cd /d %USERPRPOFILE%\Pictures(进入图片目录)
输入start . （注意start和点之间有空格）

```



```bash
 
 
copy:复制
move:移动
C:\(根目录)：通常有Windows（机房）、Program Files(软件安装区)、Users（住户区）
C:\Users(用户区)
C:\Users\win16(你的home)：包含Desktop、Documents、Downloads、Pictures、Music、Videos

例1：把一个文件从“文档”隔空传送到“桌面”（Desktop)

在文档里造一个文件
去“文档”
cd /d %USERPROFILE%\Decuments
造一个机密文件
echo 这是一个机密文件 > secret.txt
确认它在文档里
dir secret.txt

执行传送
要用move命令，逻辑是move[要搬谁][搬去哪]
例：把secret.txt 搬运到桌面
move secret.txt %USERPROFILE%\Desktop

```



```bash

数据重定向
比如你运行命令（如ipconfig),结果是“打印”在屏幕上的，关闭窗口就没了，但在服务器管理中，我们需要把这些结果“保存”下来做日志
用到重定向符（>)
注意：单箭头>(覆盖模式):如果文件不存在，它创建；如果文件例原本有字，他会直接清空原来的内容，只留新的
双箭头>>(追加模式)：它会在文件末尾接着写，保留原来的内容
例：把网络详细信息，直接保存到桌面上，做成一份报告
1、确保我们先去桌面
cd /d %USERPROFILE%\Desktop
2、执行命令，但把结果进文件
ipconfig /all > Network_Report.txt
3、验证文件是否生成
dir Network_Report.txt
4、直接用记事本打开它
notepad Network_Report.txt

例：我们试着往刚才的报告里，再追加一行时间，证明它不会覆盖
1、把当前时间”追加“到报告末尾
time /t >> Network_Report.txt
2、再次打开检查
notepad Network_Report.txt
```



```bash
通配符
*代表：所有东西/任意长度的字符

例：我们先制造一点”混乱“，然后再一次性清理掉
1、制造3个垃圾文件
echo test > a.log
echo test > b.log
echo test > c.log

2、确认他们存在
dir *.log

3、精准打击：只删除.log结尾的文件
del *.log

4、再次确认，垃圾清理干净了
dir *.log

```



```bash
自动化脚本
批处理脚本（.bat):他就是一个普通的文本文件，知识后缀改成了.bat，Windows看到这个后缀，就知道这里里面的每一行字，要当成命令去执行

例：编写一个”一键网络体检“工具
1、创建脚本文件请在CMD里输入：
notepad MyTool.bat
这回直接打开一个空白的记事本，文件名叫MyTool.bat
(系统会问你是否创建新文件，点”是“)
2、写入代码，把下面的复制粘贴到哪个记事本里，然后保存（ctrl+s）并关闭记事本
```

```bash
@echo off
:: 上面这句是让黑窗口别废话，只显示结果

echo ===============================
echo 正在启动架构师自动巡检工具...
echo ===============================

echo.
echo 正在查询网络配置，请稍候...
ipconfig /all > Auto_Report.txt

echo.
echo 正在追加时间戳...
date /t >> Auto_Report.txt
time /t >> Auto_Report.txt

echo.
echo 任务完成！报告已生成：Auto_Report.txt
echo ===============================

:: 暂停一下，不然窗口一闪就没了，你看不清结果
pause
```

```basg
现在回到CMD黑窗口，只需要输入这个脚本的命令
MyTool.bat
桌面上会多了一个Auto_Report.txt
```



```bash
PowerShell脚本

上面的.bat脚本虽然能用，但是他是一个只能处理死板的文字，不能做数学题，不能弹窗，也不能很容易地分辨红绿颜色

例：要创建一个.ps1（PowerShell Script)脚本，它不仅能查信息，还能调用Windows的核心功能
1、重新打开一个窗口：按win键，搜PowerShell,右键-》以管理员身份运行，输入下面的命令
查看当前的策略（通常是Restricted - 受限）
Get-ExecutionPolicy
解除封印（允许运行本地写的脚本）
Set-EXecutionPolicy RemoteSigned
系统会问你是否更改，请输入Y并回车
2、先回桌面
cd $env:USERPROFILE\Desktop
创建脚本文件
New-Item -Path "SuperTool.ps1" -ItemType File

现在，右键点击桌面上新出现的SuperTool.ps1,选择”编辑“或用记事本打开。（如果有一堆选项，选“Windows PowerShell ISE"体验最好，或者记事本也行）
把下面的这段代码粘贴进入，保存并关闭
```

```bash
# === Super Tool Fixed ===

Write-Host "Starting Scan..." -ForegroundColor Cyan

$now = Get-Date
Write-Host "Time: $now" -ForegroundColor Yellow

# 检查网络
$net = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }

if ($net) {
    Write-Host "OK: Network is Up!" -ForegroundColor Green
    [System.Console]::Beep(1000, 200)
} else {
    Write-Host "ERROR: Network Down!" -ForegroundColor Red
    [System.Console]::Beep(500, 500)
}

Read-Host "Press Enter to exit..."
```

```basgh
3、运行它
.\SuperTool.ps1
```

  