- [vim指南](#vim指南)
  - [复制](#复制)
  - [删除](#删除)
  - [粘贴](#粘贴)
  - [撤销](#撤销)
  - [查找 & 替换](#查找--替换)
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
