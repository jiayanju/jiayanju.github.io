---
title: "Git工作常用命令记录" 

date: 2021-03-21T21:30:00+08:00 

categories: ["git"]

tags: ["git","工作效率"] 

author: "Me" 

showToc: true

TocOpen: false 

draft: false 

hidemeta: false 

comments: false 

description: "" 

disableHLJS: true  

disableShare: false 

cover:
  image: "https://raw.githubusercontent.com/jiayanju/imgrepo/main/git.jpg"
  alt: ""
  caption: ""
  relative: false
  hidden: false

---

git的基本命令就不在赘述，这里主要是记录一下工作中常用或者有用但不常用的命令，希望可以帮助到别的同学。

## 1. 合并代码

git有俩种方式可以把更改从一个分支放到另一个分支，merge和rebase。使用merge的话，会在被放到的分支上生成一个新的commit记录，最后会使分支的commit历史记录变得不清晰明了。所以推荐使用rebase的方式来合并分支，所有的commit历史记录都在一个分支线上，查找起来也比较容易。

#### 1. rebase的常规使用

通常我们可以这样使用rebase

```bash
$ git checkout -b feature_xx_branch   -- 创建并切换到feature分支
... -- 开发代码
$ git add .
$ git commit -m '实现XX功能'

$ git rebase develop
... -- 合并冲突等然后执行git rebase --continue
$ git checkout develop
$ git merge feature_xx_branch  -- 已经rebase过了，merge的时候不会生成单独的合并commit
```

#### 2. 使用rebase合并commit

在功能开发的时候，我们在自己的feature分支上经常会commit多次，因此在feature分支上会有多个commit记录。但合并到develop的时候，为了develop分支的提交历史记录更清晰，需要这个feature只要一个commit记录。这就需要把feature分支上的多个commit合并为一个commit。

可以使用git rebase -i 命令来做交互式的修改（git rebase --interactive），这个命令需要在后面指定修改的commit范围，有多少个commit需要修改，比如feature分支上有5个commit记录需要合并为1个，使用HEAD~5来指定范围。

```bash
$ git rebase -i HEAD~5
```

执行完上面的命令后，会出现如下的提示:

```
pick 1fb69cd update
pick e8ec7b0 update
pick 81f5262 添加文章
pick 88283a8 update
pick 7307eb2 publish

# Rebase 3334dbb..7307eb2 onto 3334dbb (5 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
```

从上面的提示可以看到，这个命令可以实现很多的功能，包括重写comit信息，删除commit，合并comit，commit排序等。这里要实现commit的合并，可以使用fixup命令，如下所示：

```
pick 1fb69cd update
f e8ec7b0 update
f 81f5262 添加文章
f 88283a8 update
f 7307eb2 publish
```

然后保存，就会自动rebase相应的commit记录到第一个记录了。实现了合并commit的功能。

其他的功能使用，可以参考下面的文档

https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History



#### 3. 合并分支上的一系列commit到另一个分支上

这个情况是这样的，在开发过程中也会遇到。

你的功能开发需要基于另一个同事的feature分支上开发。然后同事的feature分支开发完成后，合并为一个commit然后合并到develop了，然后你的feature分支同时还需要develop分支里面其他人的commit。这种情况如下：

![](https://raw.githubusercontent.com/jiayanju/imgrepo/main/git-rabse-onto.png)

如图片里面所示

feature_xx是同事的分支，之后合并到develop为X'的commit

feature_yy是你的分支，需要基于最新的develop分支。

可以使用如下的rebase命令实现：

```
git checkout -b <new_branch_name> <to-commit-id>
创建一个新的分支，指明新分支的最后一个commit

git rebase --onto <branch_name> <from-commit-id>
rebase这个分支到要合并到的分支上面，branch_name即为要合并到的分支如develop或者master等。指定从哪一个commit开始
```

对应图示里面的命令如下

```
$ git checkout -b feature_yy_new Y3

$ git rebase --onto develop Y1^
```

其他rebase --onto的合并分支例子可以参考下面的文档

https://git-scm.com/book/en/v2/Git-Branching-Rebasing

#### 4. Cherry-Pick

Cherry-Pick就比较简单了，可以实现把一个commit合并到另一个分支上。

```
$ git cherry-pick <commit-id>
```

## 2. 一些记录

#### 分支重命名

git branch -m old new

#### commit作者更改

最后一个commit修改：

git commit --amend --author="NewAuthor NewEmail@address*.*com"

git push -f

非最后一个commit修改：

git rebase -i commit_id（第N+1条记录的commitId）

git commit --amend --author="NewAuthor NewEmail@address*.*com"

git rebase --continue

git push -f

#### commit时在vim界面取消操作

有的时候commit的时候，到vim界面的时候发现comit有的问题，想撤销这个操作，使用q或者q！都不影响命令的继续执行。可以使用cq命令返回

```
:cq
```

#### 撤回commit

git reset --soft HEAD^

HEAD^的意思是上一个版本，也可以写成HEAD~1
如果你进行了2次commit，想都撤回，可以使用HEAD~2

`--mixed `
意思是：不删除工作空间改动代码，撤销commit，并且撤销git add . 操作
这个为默认参数，`git reset --mixed HEAD^` 和 `git reset HEAD^` 效果是一样的。

`--soft` 
不删除工作空间改动代码，**撤销commit**，**不撤销`git add .`** 

`--hard`
**删除工作空间改动代码**，**撤销commit，撤销`git add .`** 
**注意完成这个操作后，会删除工作空间代码！！！恢复到上一次的commit状态。慎重！！！**

#### 撤回git commit --amend

有的时候需要commit代码却手滑使用了amend，怎么撤销呢。

首先使用git reflog

```
2148246 (HEAD -> main) HEAD@{0}: commit (amend): update1
5bbf58f HEAD@{1}: commit: update
7307eb2 (origin/main) HEAD@{2}: checkout: moving from test to main
7307eb2 (origin/main) HEAD@{3}: checkout: moving from main to test
7307eb2 (origin/main) HEAD@{4}: commit: publish
88283a8 HEAD@{5}: commit: update
81f5262 HEAD@{6}: pull: Fast-forward
e8ec7b0 HEAD@{7}: commit: update
1fb69cd HEAD@{8}: commit: update
3334dbb HEAD@{9}: pull: Fast-forward
08108cb HEAD@{10}: commit: update
62a4474 HEAD@{11}: pull --rebase origin main: Fast-forward
6e90f11 HEAD@{12}: commit: publish config changes
2fb237c HEAD@{13}: commit: config change
```

可以看到里面记录各种操作记录

然后使用

```bash
git reset --soft HEAD@{1}
```

就可以了

