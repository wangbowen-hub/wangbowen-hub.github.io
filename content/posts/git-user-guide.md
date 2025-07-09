---
title: "Git User Guide"
date: 2025-07-09T15:37:25+08:00
# bookComments: false
# bookSearchExclude: false
---

# Git使用记录

## .gitignore

* 如果文件在添加到 .gitignore 之前已经被 Git 追踪，那么即使添加到 .gitignore 中，Git 仍会继续追踪它
* 切换分支时，.gitignore 文件的内容会切换到该分支的版本



## git rm

git rm 用于从 Git 仓库中删除文件，它会同时从 Git 索引（暂存区）和工作目录中删除文件

* -f 强制删除
* --cached 只从 Git 索引中删除，保留工作目录中的文件，文件变成"未跟踪"状态，常用于将已提交的文件加入 .gitignore
* -r 删除整个目录及其内容，与其他参数组合使用



## git checkout

* git checkout --ours files 选择本分支版本文件解决冲突 

  * --ours：指当前分支（你执行 merge 命令时所在的分支）

  - --theirs：指要合并进来的分支

* -b 创建并切换到该分支

## git ls-files
查看已追踪的文件
