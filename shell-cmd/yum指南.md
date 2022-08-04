- [yum指南](#yum指南)
  - [rpm](#rpm)
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
