### 1、撤回commit中的提交（未push）
```git
git reset --mixed <cmomit ID> 或者git reset (Head^)<commitID>     #撤销未push的代码，保留本地修改
git reset --soft (HEAD^)<commitID>        #撤销commit 但是不撤销add
git reset --hard            #直接回退上一个版本本地修改也不保留
git reset Head <file> or .  #从暂存区撤出
git rm --cached *(文件名)   #删除缓冲区中的文件

git log --pretty=oneline #输出一行
git reflog             #记录命令 可以查看被回退了的版本然后结合
git show commit_id     #查看指定提交commit的内容
git log --author=""    #查看某个人提交的日志

git log 查看提交日志，
git checkout my_branch #切换到指定分支
git cherry-pick commit_id  #从其它分支提交的合并到当前分支

git merge 源分支 目标分支

git branch -d 分支;git push origin -d 分支  #删除分支
```

### 2、git stash
```git
git stash 备份当前工作区的内容，保存到git栈中，从最近的一次commit中读取相关内容
git stash list 查看所有被隐藏的文件列表
git stash pop 获取最近的一次数据并且恢复工作区
git stash apply stash@{1} 获取对应的备份，但是不会删除
git stash drop stash@{} 删除对应的
```

### 3、git命令
![image](E:\文档\note\imgs\18087435-bf2a996ef50a21b0.jpg)