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
- [系统管理](#系统管理)
  - [进程信息](#进程信息)
  - [磁盘信息](#磁盘信息)
  - [系统信息](#系统信息)
  - [常用命令](#常用命令)
    - [重定向](#重定向)
- [yum指南](#yum指南)
  - [rpm](#rpm)
- [vim指南](#vim指南)
  - [复制](#复制-1)
  - [删除](#删除)
  - [粘贴](#粘贴)
  - [撤销](#撤销)
  - [查找 & 替换](#查找--替换)
- [Bash](#bash)
  - [数据显示](#数据显示)
  - [变量赋值](#变量赋值)
    - [字符串拼接](#字符串拼接)
  - [数据计算](#数据计算)
    - [浮点运算](#浮点运算)
  - [语法](#语法)
    - [if 语句](#if-语句)
      - [数值比较](#数值比较)
      - [字符串比较](#字符串比较)
      - [文件比较](#文件比较)
    - [循环语句](#循环语句)
      - [for](#for)
      - [while](#while)
    - [输入](#输入)
      - [输入参数](#输入参数)
      - [移动参数](#移动参数)
      - [输入选项](#输入选项)
        - [选项标准化](#选项标准化)
      - [用户输入](#用户输入)

```bash

```
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
# yum指南
```bash
# 列出已安装的程序
yum list installed
```

```bash
# 从本地安装rpm格式的包
yum localinstall [package_name.rpm]
```

```bash
# 列出所有可以更新的程序
yum list updates
```

```bash
# 更新所有可以更新的程序
yum update
```

## rpm
```bash
# 显示以安装的软件包
rpm -qa
```

# vim指南
## 复制
```bash
yw 复制一个单词
y$ 复制一行内容
```
## 删除
```bash
dd 删除当前光标所在行
```
## 粘贴
```bash
p 粘贴(同时也会将删除的内容打印到当前光标处)
```
## 撤销
```bash
u 撤销前一编辑命令
```
## 查找 & 替换
```bash
/ [str] 查找str
n 在上面的基础上按n表示查找下一个str
```
```bash
:s/hello/c++,hello  将单个hello替换成c++,hello
:s/hello/c++,hello/g  将所有hello替换为c++,hello
:n,ms/hello/c++hello/g 替换n和m行之中的所有hello
```

# Bash
## 数据显示
```bash
# 打印str，不带换行符
echo -n [str]
```
## 变量赋值
```bash
# 将cmd命令的输出赋值给var
var=$(cmd)
```
### 字符串拼接

```bash
# 注意不要有空格
str1="hello"
str2=$str1" world"
```

## 数据计算
```bash
# 将var_a和var_b经过数据计算的结果赋值给int_var
# 注：bash只支持整数运算
int_var=$[ var_a operator var_b ]
```
### 浮点运算
```bash
# 通过bc命令支持浮点运算
var=$(echo "scale=4; 3.5/2" | bc)
```
## 语法

### if 语句
```bash
# test命令会测试var之中是否有内容
if [condition]
then
  [cmd]
elif [condition]
then
  [cmd]
elif test $var
then
  [cmd]
else
  [cmd]
fi
```
#### 数值比较
```bash
n1 -eq n2 检查n1是否与n2相等
n1 -ge n2 检查n1是否大于或等于n2
n1 -gt n2 检查n1是否大于n2
n1 -le n2 检查n1是否小于或等于n2
n1 -lt n2 检查n1是否小于n2
n1 -ne n2 检查n1是否不等于n2
```
#### 字符串比较

```bash
str1 = str2 检查str1是否和str2相同
str1 != str2 检查str1是否和str2不同
str1 < str2 检查str1是否比str2小
str1 > str2 检查str1是否比str2大
-n str1 检查str1的长度是否非0
-z str1 检查str1的长度是否为0
```
#### 文件比较

```bash
-d file 检查file是否存在并是一个目录
-e file 检查file是否存在
-f file 检查file是否存在并是一个文件
-r file 检查file是否存在并可读
-s file 检查file是否存在并非空
-w file 检查file是否存在并可写
-x file 检查file是否存在并可执行
-O file 检查file是否存在并属当前用户所有
-G file 检查file是否存在并且默认组与当前用户相同
file1 -nt file2 检查file1是否比file2新
file1 -ot file2 检查file1是否比file2旧
```
### 循环语句
#### for

```bash
# list之中各元素空格处理，如果单独的元素之中有空格需要用""将这个元素圈起来
for var in [list]
do
  commands
done
```
#### while

```bash
# list之中各元素空格处理，如果单独的元素之中有空格需要用""将这个元素圈起来
var=10
while [ $var -gt 0 ]
do
  echo $var
  var=$[$var - 1]
done
```
### 输入
#### 输入参数

```bash
# 记录输入参数的个数
$#

# 一次性获取所有输入参数
$@

# 脚本名称
$0

#第一个参数
$1
```
#### 移动参数

```bash
# shift会向右遍历参数，也就是$1此时作为第一个参数变量之后，会作为第二个参数变量...
count=0
while [ -n "$1" ]
do
  echo "Parameter #$count = $1"
  count=$[ $count + 1 ]
  shift
done
```
#### 输入选项

```bash
# 输入./test.sh -a test
while [ -n "$1" ]
do
  case "$1" in
    -a) echo "Found the -a option" ;;
    -b) echo "Found the -b option" ;;
    -c) echo "Found the -c option" ;;
    *) echo "$1 is not an option" ;;
  esac
  shift
done
```
##### 选项标准化

```bash
-a 显示所有对象
-c 生成一个计数
-d 指定一个目录
-e 扩展一个对象
-f 指定读入数据的文件
-h 显示命令的帮助信息
-i 忽略文本大小写
-l 产生输出的长格式版本
-n 使用非交互模式（批处理）
-o 将所有输出重定向到的指定的输出文件
-q 以安静模式运行
-r 递归地处理目录和文件
-s 以安静模式运行
-v 生成详细输出
-x 排除某个对象
-y 对所有问题回答yes
```
#### 用户输入

```bash
# read将用户输入放入变量name之中
read name
```
