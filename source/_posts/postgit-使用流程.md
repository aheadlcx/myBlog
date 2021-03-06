title: git 使用流程
date: 2016-01-18 15:44:55
tags: [git, 工作流, 笔记]
categories: git
---
记录git个人操作实践。多用命令行，少用GUI，可以有助理解git的工作原理。
<!--more  -->

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="http://music.163.com/outchain/player?type=2&id=5268118&auto=1&height=66"></iframe>


# git远程仓库
我们一般clone下来的仓库，默认都是origin的，其实git是可以支持多远程仓库的。
1. git remote -v 是可以查看本地库，有那些远程库。
2. git remote add [shortname] [url] 这是添加远程库。  
例如git remote add pd  https://github.com/xiongwei-git/Inputer  
3. git fetch pd 意思就是把远程库pd，所有分支都拉到本地，如果远程库pd分支master，  
那么对应的本地分支，就是pd/master. 如果需要用到此分支，需要自行自行合并。最后删除本地分支。
4. git remote show [shortname] 这个是查看远程仓库的信息。[shortname] 代表的是远程仓库。
5. git remote rename oldshortname newshortname 把远程仓库改名字。
6. git remote rm [shortname] 删除远程仓库。
7. git branch -a  查看远程库分支信息。

# git 日常的操作
1. git push [shortname] localBranch:remoteBranch 这是把本地分支推送到远程分支的格式  
其实就是 git push 仓库名 本地分支:远程分支
2. git pull [shortname] localBranch:remoteBranch 同上。
3. git push [shortname] :remoteBranch 删除远程分支
4. git checkout -b localBranch shortname/remoteBranch 拉取远程分支到本地，并且本地分支  
跟踪远程分支。这个好处就是，以后直接调用git push和git pull，就可以直接推送和拉取代码了。  
其实我们一般clone下来的仓库，都是自动创建了master分支，并且跟踪远程的master分支。
5. git reset HEAD file 这是把放到暂存区的文件，拿出来。
6. git checkout --file 这是把工作区的内容撤销。
7. git diff 查看修改内容
8. git reset --hard HEAD^  恢复到上一个版本
9. git reset --hard commitId 恢复到特定版本
10. git push --set-upstream origin branchName     设置本地分支的跟踪分支
11. git push origin --tags 推送所有tag到仓库.相应改为 git pull 则是拉取tag下来
12. git pull --rebase origin dev 以变基的方式来同步 origin 的 dev 分支。



# git 紧急修bug场景相关操作
1. git stash 此命令可以保存现场环境，可以切换到其他分支，继续工作，以后再切换回来此分支，可以再回复  
现场环境。
2. git stash pop 恢复现场环境，并且删除现场。
3. git stash list 是查看有哪些保存的现场
4. git stash apply 是恢复现场，但是不删除
5. git stash drop 是删除现场。
6. git stash pop stash@{1} 可以把编号为 1  的记录，恢复并且删除。

# git工作流
这是本人工作中用到的一个工作流，简单介绍下。
1. 项目库主要有master release dev feature分支。项目库，一般的开发者，不应该有写入权限。
2. 每个开发者fork 项目库，并且拉取到本地仓库。并且设置个人库一直跟随同步项目库。  
Stash 带有这个功能。其实也可以，在每次push到个人库前，先从项目库拉取dev分支到本地库dev分支。  
再push本地分支dev到个人库dev。
3. 开发者在自己的feature A分支上开发。
4. 开发完，先拉取个人库的dev分支到本地dev分支，然后合并dev分支到个人feature A分支，  
5. 推送feature A分支到个人库。并且pull request feature A 分支到项目库 的公共feature C分支。  
这个公共featuer分支，可以按照迭代版本来设定。
6. leader，review 代码，审核开发者提交的pr。merge feature A 分支到 公共feature C 分支。
7. 然后leader，merger 公共feature C分支到dev分支。
8. 每次把功能基本做完的时候，就从dev拉取出来一个分支release，提交给测试组。在这之后，从release拉取代码，开展bug分支作业。稳定可以发布之后，就从release merge到dev分支。再merge dev分支到master分支。以供发布。

# 工作中遇到的问题
1. merge的时候，提示让你输入commit messag
那个界面是个vim模式  
i 是insert
e 是edit
a 是append
:wq 是保存并退出。
多按几次esc，就是退出到阅读模式。
按e，i，a，都可以进入到vim的编辑模式，光标位置不一样。
2.  如果在 feature 分支提交了，但是需要在 master 分支上应用此修改，因此可以
选择 git cherry-pick commitId , 有可能会产生冲突，add commit 即可。

# 撤销相关的操作
1. revert 恢复
提交了一个 commit ，版本回退是可以，但是如果希望在 commit history 看到这个操作的话，更好  
的选择是 git revert commitId ，这样会做一个相反的操作。
2. checkout
git checkout -- filename 把工作内容撤销
3. git rebase
首先，它定位你当前检出分支和master之间的共同祖先节点（common ancestor）。  
然后，它将当前检出的分支重置到祖先节点（ancestor），并将后来所有的提交都暂存起来。  
最后，它将当前检出分支推进至master末尾，同时在master最后一次提交之后，  
再次提交那些在暂存区的变更。  
rebase 如果有冲突的话，需要自行解决，并且 add ，之后就 rebase -- continue 。而不是 commit
4. git rebase -i commitId
场景： 你开始朝一个既定目标开发功能，但是中途你感觉用另一个方法更好。你已经有十几个提交，  
但是你只想要其中的某几个，其他的都可以删除不要。  
这个命令会打开 文本编辑器，然后可以删除不想要的 commitId ，保留剩下的 commitId 。
5. 停止跟踪一个已经跟踪的文件
如果一个文件已经被添加到 git 中，现在再加进去 gitIgnore ，这个是没用的。其中一个办法是  
先删除了，再添加进去。另外一个办法是，git rm -cached fileName ，并且不会在磁盘上删除  
该文件。

# submodule
添加子模块：$ git submodule add [url] [path]
如：$ git submodule add git://github.com/soberh/ui-libs.git src/main/webapp/ui-libs
初始化子模块：$ git submodule init ----只在首次检出仓库时运行一次就行
更新子模块：$ git submodule update ----每次更新或切换分支后都需要运行一下
删除子模块：（分4步走哦）
+ $ git rm --cached [path]
+  编辑“.gitmodules”文件，将子模块的相关配置节点删除掉
+ 编辑“.git/config”文件，将子模块的相关配置节点删除掉
+ 手动删除子模块残留的目录

> [git 子模块使用](http://easior.is-programmer.com/posts/42541.html)
# git book 文档
这个比较完整的官方文档
http://git-scm.com/book/zh/v2
