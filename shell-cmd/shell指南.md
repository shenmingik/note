- [文件系统](#文件系统)
  - [显示](#显示)
  - [复制](#复制)
  - [链接](#链接)
  - [目录](#目录)
  - [查看文件](#查看文件)
  - [文件操作](#文件操作)
    - [排序](#排序)
    - [筛选](#筛选)
  - [压缩](#压缩)
- [命令行编辑器](#命令行编辑器)
  - [sed](#sed)
    - [行替换](#行替换)
    - [删除行](#删除行)
    - [添加行](#添加行)
    - [修改行](#修改行)
    - [打印行](#打印行)
  - [awk](#awk)
    - [从命令行读取程序脚本](#从命令行读取程序脚本)
    - [读取文件](#读取文件)
    - [文本替换](#文本替换)
- [系统管理](#系统管理)
  - [进程信息](#进程信息)
  - [磁盘信息](#磁盘信息)
  - [系统信息](#系统信息)
  - [常用命令](#常用命令)
    - [重定向](#重定向)

# 文件系统

## 显示
```bash
# 显示所有文件的所有详细信息
ls -alF [file]
```

## 复制
```bash
# -i表示当同名文件存在的时候，shell会进行询问
cp -i [source] [destination]
```
```bash
# 递归复制整个文件夹
cp -R [source] [destination]
```

## 链接
```bash
# 创建软链接
ln -s [source] [new_file]
```

## 目录
```bash
# 如果dir1，dir2不存在也可以创建
mkdir -p [dir1/dir2/dir3]
```

## 查看文件
```bash
# 查看文件的时候添加行号信息
cat -n [file]
```

```bash
# 显示后面n行内容
tail -[n] [file]
```

```bash
# 显示头部n行内容
head -[n] [file]
```

```bash
# 查看第n行的内容
sed -n 'np' [file]
```

## 文件操作
### 排序
```bash
# 文件之中按照数值排序，字符不可用
sort -n [file]
```

```bash
# 逆序排序
sort -r [file]
```

```bash
#  -k指定参与操作的字段n，和-t参数配置,-t参数用于按照特定字符分隔字符串
sort -t ':' -k [n] [file]
```

### 筛选
```bash
# 过滤不出现这个str的数据
grep -v [str] [file]
```

```bash
# 过滤时显示行号
grep -n [str] [file]
```

```bash
# 显示满足str1，str2两个条件的行
grep -e [str1] -e [str2] [file]
```

## 压缩
```bash
# 压缩
tar -cvf [zip] [files]
```

```bash
# 解压
tar -xvf [zip] [path]
```
# 命令行编辑器
## sed
sed基本工作原理如下：

(1) 一次从输入中读取一行数据。

(2) 根据所提供的编辑器命令匹配数据。

(3) 按照命令修改流中的数据。

(4) 将新的数据输出到STDOUT。
```bash
# sed命令格式
sed [options] [command] [file]

# 使用多个命令
sed -e [command] -e [command] [file]
```

### 行替换

```bash
# 格式如下
# begin：指定开始替换行号
# end：指定结束替换行号
#
# 有四种替换flag:
# 数字：表面文本中每行的第几处匹配位置将被替换
# g：表面新闻本将会替换所有匹配文本，这也是默认处理方式
# p：表面原先行的内容要打印出来
# w file：表面将食醋胡结果写到file文件之中
'[begin,end]s/[pattern]/[replacement]/[flags]'
```
```bash
# 以下命令将[2,3]闭区间的内容中的dog替换为cat
# [2,$]可以表示第二行开始之后的所有行
# [$,$]可以表示所有行
sed '2,3s/dog/cat/' test.txt
```

```bash
# 下面的代码之中将"test"文本替换为"big test"
echo "replace test to big test" | sed 's/test/big test/'
```
```bash
# 多个替换命令之间使用分号(;)进行分割
echo "replace test to big test" | sed 's/test/big test/; s/replace/compare/'
```

### 删除行
```bash
# 格式:
sed '[begin,end]d' file
```

```bash
# 下面的命令用于删除[2,3]范围内的行
sed '2,3d' test.txt
```

```bash
# 下面的命令用于删除包含字符"number 1"的行
sed '/number 1/d' test.txt
```
### 添加行
```bash
# 格式
sed '[line_num]i\[str]' file
```

```bash
# 下面的命令用于在test.txt文件之中第三行的位置插入一行内容
sed '3i\add a new line' test.txt
```
### 修改行

```bash
# 格式
sed '[line_num]c\[str]' file
```

```bash
# 下面的命令用于将test.txt文件之中第三行修改为change a line
sed '3c\change a  line' test.txt
```

```bash
# 下面的命令用于将test.txt文件之中包含"old line"的行修改为"change a line"
sed '/old line/c\change a  line' test.txt
```
### 打印行
```bash
# 基本格式
sed -n '[begin,end]p' file
```

```bash
# 下面的命令用于打印test.txt文件的[1,5]行
sed -n '1,5p' test.txt
```

## awk
awk用于执行用户指定的代码，当编写好代码之后摁下Enter之后，其会等待用户输入然后执行刚刚写好的代码。
```bash
# 基本格式
awk [options] [program] [file]
```
### 从命令行读取程序脚本
```bash
# 下面的命令会等待用户输入，用户每输入一次就会打印一次hello world
awk '{print "hello world!"}'
```

```bash
# BEGIN关键字使程序先执行程序脚本，而不必等待用户输入
# 同样，END关键字允许在读取完输入数据之后执行程序脚本
# 注：可以理解为构造函数和析构函数
awk 'BEGIN{print "hello world!"}'

echo "hello world!" | awk '{print $0} END{print "hello world1!"} {print $0}'
```


### 读取文件
```bash
# awk将每行的字段按照空格拆分，其中:
# $0 表示整行
# $1 表示每行第一个元素
# $n 表示每行第n个元素

# 下面的命令打印每行的第一个元素
awk '{print $1}' test.txt
```

```bash
# -F参数用于指定其他分隔符
awk -F: '{print $1}' test.txt
```
### 文本替换

```bash
# 以下命令将本文的第三个字段做替换并打印
echo "today is 2022.8.4" | awk '{$3 = "2022.8.3"; print $0}'
```
# 系统管理
## 进程信息
```bash
# 显示所有进程的详细信息
ps -el
```

## 磁盘信息
```bash
# 显示路径下占用的磁盘信息
du -csh [path]
```

## 系统信息
```bash
# 显示时间
date

# 以年月日的方式显示时间
data +%y%m%d
```
```bash
# 显示登录用户
who
```

## 常用命令
### 重定向
```bash
# 以覆盖的方式将cmd的命令输出转到file之中
[cmd] > [file]
```
```bash
# 以追加的方式将cmd的命令输出转到file之中
[cmd] >> [file]
```