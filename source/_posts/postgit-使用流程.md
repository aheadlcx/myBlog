title: git 使用流程
date: 2016-01-18 15:44:55
tags: git
categories: git
---

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

# git 日常的操作
1. git push [shortname] localBranch:remoteBranch 这是把本地分支推送到远程分支的格式  
其实就是 git push 仓库名 本地分支:远程分支
2. git pull [shortname] localBranch:remoteBranch 同上。
3. git push [shortname] :remoteBranch 删除远程分支
4. git checkout -b localBranch shortname/remoteBranch 拉取远程分支到本地，并且本地分支  
跟踪远程分支。这个好处就是，以后直接调用git push和git pull，就可以直接推送和拉取代码了。  
其实我们一般clone下来的仓库，都是自动创建了master分支，并且跟踪远程的master分支。

# git 紧急修bug场景相关操作
1. git stash 此命令可以保存现场环境，可以切换到其他分支，继续工作，以后再切换回来此分支，可以再回复  
现场环境。
2. git stash pop 恢复现场环境，并且删除现场。
3. git stash list 是查看有哪些保存的现场
4. git stash apply 是恢复现场，但是不删除
5. git stash drop 是删除现场。
