
# Git指南

## 常用类

## 分支
```bash
# 该命令将显示当前代码库中所有的本地分支
git branch

# 该命令将创建一个分支
git branch -d [branch name]

# 切换分支
git checkout [branch name]

# 创建并切换
git checkout -b [branch name]

# 该命令可以将指定分支的历史记录合并到当前分支
git merge [branch name]
```
## 临时保存文件
```bash
# 该命令将临时保存所有修改的文件
git stash save

# 该命令将恢复最近一次stash（储藏）的文件
git stash pop

# 该命令将显示stash的所有变更
git stash list

# 该命令将丢弃最近一次stash的变更
git stash drop
```

## stage

### 显示待stage的文件
```bash
# 该命令将显示所有需要提交的文件
git status
```

### 撤销stage
```bash
# 该命令将从stage中撤出指定的文件，但可以保留文件的内容
git reset [file]

# 该命令可以撤销指定提交之后的所有提交，并在本地保留变更
# git reset HEAD就是撤销add
git reset [commit_ptr]

# 该命令将丢弃所有的历史记录，并回滚到指定的提交
git reset –hard [commit_ptr]

# 撤销commit但是保留add
git reset --soft HEAD^
```

### 版本差异

```bash
# 该命令可以显示尚未添加到stage的文件的变更
git diff

# 该命令可以显示添加到stage的文件与当前最新版本之间的差异
git diff –staged

# 该命令可以显示两个分支之间的差异
git diff [first branch] [second branch]
```

## 用户信息设置
```bash
# 设置提交代码的用户名
git config –global user.name "[user_name]"

# 设置提交代码的电子邮箱地址
git config –global user.email "[user_email]"
```

##