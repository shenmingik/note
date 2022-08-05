- [GDB调试指南](#gdb调试指南)
  - [CoreDump文件](#coredump文件)
  - [正在运行文件](#正在运行文件)
  - [运行相关命令](#运行相关命令)
  - [设置断点](#设置断点)
  - [线程切换](#线程切换)
  - [查看源码](#查看源码)
  - [查看运行信息](#查看运行信息)
  - [监视](#监视)
  - [分割窗口](#分割窗口)


# GDB调试指南

## CoreDump文件
```bash
# 运行coredump文件
gdb [program] [core_file]
```

## 正在运行文件
```bash
gdb [program] [pid]
```

## 运行相关命令
```bash
# 运行程序，当遇到断点后，程序会在断点处停止运行
run(r)

# 继续执行，到下一个断点处
continue(c)

# 单步跟踪程序，当遇到函数调用时，也不进入此函数体
next(n)

# 单步调试如果有函数调用，则进入函数
setp(s)

# 运行程序直到某行
until [line]

# 调用程序中可见的函数，并传递参数
call [function[param]]

# 退出
quit
```

## 设置断点
```bash
# 在第line行设置断点
break [line]

# 在函数入口点设置断点
break [function]

# 删除第n个断点
delete [break_point_num]

# 清除所有断点
delete breakpoints
```

## 线程切换
```bash
# 切换到线程id
thread [id]
```

## 查看源码

```bash
# 每次显示10行源代码
list(l)

# 将显示当前文件以“行号”为中心的前后10行代码
list [line]

# 显示函数所在的源代码
list [function]
```

## 查看运行信息

```bash
# 查看当前运行的堆栈列表
bt

#  n是一个从0开始的整数，是栈中的层编号。比如：frame 0，表示栈顶，frame 1，表示栈的第二层。
frame [n]

# 指定运行时参数
set [args]

# 查看变量
print [var]

# 显示当前堆栈的所有变量
info locals

# 查询函数
info function

# 查看所有线程
info threads
```

## 监视
```bash
# 设置一个监视点，一旦被监视的“表达式”的值改变
watch [var]
```

## 分割窗口
```bash
# 显示源代码窗口
layout src

# 显示反汇编窗口
layout asm

# 刷新窗口 
ctrl+L
```