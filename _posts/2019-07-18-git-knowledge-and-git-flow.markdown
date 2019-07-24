---
layout:     post
title:      "git基础和git flow"
subtitle:   "git knowledge and git flow"
date:       2019-07-18
author:     "David"
header-img: "img/tag-bg.jpg"
tags:
    - git
---



### 背景

git由Linus主导开发，用于程序员的项目协作。


### 原理

本地git仓库分为工作区（文件区域）、暂存区(stage)和HEAD组成。

![git repo](/img/in-post/git-knowledge-and-git-flow/git.jpg)
<small class="img-hint">git仓库</small>


### 命令

#### 添加和提交

```git
# 创建仓库并初始化（生成master主分支）
git init

# 将工作区文件的快照添加到git暂存区
git add <filename>

# 提交快照到所在分支
git commit -m "commit message"
```

#### 推送改动

```git
# 添加远程仓库链接
git remote add origin <remote_git_link>

# 推送到远程仓库的特定分支
git push -u origin <branch>

# 重命名远程仓库，一般无需此操作，默认远程仓库名为origin
git remote rename <new_name> <old_name>

# 查询远程信息
git remote -v
```

#### 分支管理

```git
# 创建新分支并切换过去
git checkout -b <branch>

# 切换到其他分支
git checkout <other_branch>

# 删除某个分支
git branch -d <branch>

# 推送新分支到远程仓库
git push origin <branch>
```

#### 更新与合并

```git
# 更新本地仓库至最新改动
git pull

# 合并其他分支到当前分支
git merge <branch>
```

#### 标签

```git
# 为特定提交创建标签
git tag <tag> <commit_ID>

# 推送标签到远程仓库
git push -u origin --tags

# 删除标签
git tag -d <tag>
```

#### 替换改动

```git
# 替换工作区的改动
git checkout -- <filename>

# 从暂存区回退到工作区
git reset HEAD <filename>

# 取消对某个文件的跟踪，更新.gitignore忽略掉目标文件
git rm --cached <filename>
git commit -m "We really don't want Git to track this anymore!"
```

#### .gitignore文件

```git
# 标识出需要git忽略跟踪的文件类型，在提交修改时git自动忽略
venv/
.idea/

*.pyc
__pycache__/

instance/

.pytest_cache/
.coverage
htmlcov/

dist/
build/
*.egg-info/

*.log
```


### git flow

git flow是多人协作的git工作规范，有master和develop两大长期分支，以及feature，release和hotfix三大临时分支。

#### 工作流程

* master作为线上稳定分支，所有commit都得打标签tag。
* develop为稳定开发分支，feature和release都基于develop创建。
* feature作为开发新特性的分支，在开发完成后必须合并到develop中，然后删除。
* release作为预发布分支，主要进行测试和修复bug等，完成后合并到master和develop中，在master上打标签tag，然后删除。
* hotfix作为修复线上bug的分支，基于master创建，完成后合并到master和develop中，在master上打标签tag，然后删除。

![git flow](/img/in-post/git-knowledge-and-git-flow/git-flow.png)
<small class="img-hint">git flow流程图</small>

#### 安装git flow工具

略

#### git flow使用

```git
# 初始化
git flow init

# 开始新feature
git flow feature start <feature>

# 完成feature
git flow feature finish <feature>

# 开始新release
git flow release start <release>

# 完成release
git flow release finish <release>

# 开始新hotfix
git flow hotfix start <hotfix>

# 完成hotfix
git flow hotfix finish <hotfix>
```


### 参考

1. [git-简易指南](http://www.bootcss.com/p/git-guide/)
2. [git教程-廖雪峰](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
3. [git flow简介](http://www.codeceo.com/article/how-to-use-git-flow.html)