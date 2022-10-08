[TOC]

# Git

## 安装Git

下载Git

[DownLoad](https://mirrors.edge.kernel.org/pub/software/scm/git/)

解压Git

```
[root@t-luhx01-v-szzb ~]# tar -xvf git-2.3.0.tar.gz
```

编译安装Git

```
[root@t-luhx01-v-szzb ~]# cd git-2.3.0
[root@t-luhx01-v-szzb git-2.3.0]# ./configure --prefix=/usr/local/git
[root@t-luhx01-v-szzb git-2.3.0]# make && make install -j 4
[root@t-luhx01-v-szzb git-2.3.0]# echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/profile
[root@t-luhx01-v-szzb git-2.3.0]# source /etc/profile
[root@t-luhx01-v-szzb git-2.3.0]# git
usage: git [--version] [--help] [-C <path>] [-c name=value]
           [--exec-path[=<path>]] [--html-path] [--man-path] [--info-path]
           [-p|--paginate|--no-pager] [--no-replace-objects] [--bare]
           [--git-dir=<path>] [--work-tree=<path>] [--namespace=<name>]
           <command> [<args>]

The most commonly used git commands are:
   add        Add file contents to the index
   bisect     Find by binary search the change that introduced a bug
   branch     List, create, or delete branches
   checkout   Checkout a branch or paths to the working tree
   clone      Clone a repository into a new directory
   commit     Record changes to the repository
   diff       Show changes between commits, commit and working tree, etc
   fetch      Download objects and refs from another repository
   grep       Print lines matching a pattern
   init       Create an empty Git repository or reinitialize an existing one
   log        Show commit logs
   merge      Join two or more development histories together
   mv         Move or rename a file, a directory, or a symlink
   pull       Fetch from and integrate with another repository or a local branch
   push       Update remote refs along with associated objects
   rebase     Forward-port local commits to the updated upstream head
   reset      Reset current HEAD to the specified state
   rm         Remove files from the working tree and from the index
   show       Show various types of objects
   status     Show the working tree status
   tag        Create, list, delete or verify a tag object signed with GPG
```



## 版本库

版本库即repository，可以简单理解为一个目录，目录中的文件都是由Git负责管理

**本地仓库**

创建本地库需要设置用户名和邮箱

```
[root@t-luhx01-v-szzb git]# git config --global user.name "luhengxing"
[root@t-luhx01-v-szzb git]# git config --global user.email "690627803@qq.com"
```

初始化版本库

```
[root@t-luhx01-v-szzb bin]# mkdir /service/git/repository -p
[root@t-luhx01-v-szzb bin]# cd /service/git/repository/
[root@t-luhx01-v-szzb repository]# git init
Initialized empty Git repository in /service/git/repository/.git/
```

添加文件到Git仓库

```
[root@t-luhx01-v-szzb git]# touch test.tt
[root@t-luhx01-v-szzb git]# git add test.tt 
[root@t-luhx01-v-szzb git]# git commit -m "only test"
[master (root-commit) 26db86b] only test
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test.tt
```

**远程仓库**

除了本地仓库，我们也可以创建一台远程服务器专门作为Git仓库，接下来就简述一下部署流程
1、安装git
2、创建git用户
3、创建证书，公钥可以通过Gitosis管理
4、初始化Git仓库 `git init --bare test.git`
5、修改仓库权限 `chown -R git.git test.git`
6、禁用git用户连接系统
7、克隆远程仓库 `git clone git@10.0.139.161:/service/test.git`

## 常用操作

**查看修改内容**

如果想要查看具体修改的内容，我们可以使用diff选项来查看，与Linux命令一样

```
[root@t-luhx01-v-szzb git]# git diff test.txt 
diff --git a/test.txt b/test.txt
index e69de29..9d8a4e7 100644
--- a/test.txt
+++ b/test.txt
@@ -0,0 +1 @@
+this is test file
```

可以看出，我们添加了一行”this is test file”

**提交修改**

确定完修改内容后，我们可以选择提交变更内容，与添加文件一致，commit之前需要通过add添加后再commit

```
[root@t-luhx01-v-szzb git]# git add test.txt 
[root@t-luhx01-v-szzb git]# git commit -m "Modify test.txt"
[master 3fcf6a5] Modify test.txt
 1 file changed, 1 insertion(+)
```

**撤销修改**

在错误修改内容后，我们会希望撤回对应的修改，这里分为两种情况：一种是只修改了文件，并未添加到暂存区，撤销能够回到和当前版本一致的状态

```
[root@t-luhx01-v-szzb git]# git checkout -- test.txt
```

另一种是修改后已经add添加到暂存区了，还没有执行commit提交，可以通过reset撤销暂存区的内容

```
[root@t-luhx01-v-szzb git]# git reset HEAD test.txt 
Unstaged changes after reset:
M	test.txt
[root@t-luhx01-v-szzb git]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   test.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

可以看到修改的内容已经退回到工作区了，撤销工作区的内容只需要再执行checkout即可

```
[root@t-luhx01-v-szzb git]# git checkout -- test.txt 
[root@t-luhx01-v-szzb git]# git status
On branch master
nothing to commit, working directory clean
```

**回退版本**

log选项能够查看我们的提交记录

```
[root@t-luhx01-v-szzb git]# git log --pretty=oneline
3fcf6a5878159e994c006db2dffc897e764ae33f Modify test.txt
837262c7ee7f7470c3b6d2f99a56c2d9737dd70f only test
```

前面长长的一串字符就是commit id，你可以理解为版本号。有时我们想要回滚到之前的版本，我们必须先知道哪个是当前版本，在Git中，HEAD表示当前版本，上一个版本就是HEAD^，上上个版本就是HEAD^^，如果回滚比较多可以写成HEAD^~50。

现在我们试着回退一个版本

```
[root@t-luhx01-v-szzb git]# git reset --hard HEAD^
HEAD is now at 837262c only test
```

如果我现在还是想要回滚之前的版本呢？是否能再闪回回去？答案是可以的，只要你知道版本大致的commit id，如果没记住commit id，我们可以通过reflog查看每一次的操作命令

```
[root@t-luhx01-v-szzb git]# git reflog
3fcf6a5 HEAD@{0}: reset: moving to 3fcf6a5
837262c HEAD@{1}: reset: moving to HEAD^
3fcf6a5 HEAD@{2}: commit: Modify test.txt
837262c HEAD@{3}: commit: only test
51de587 HEAD@{4}: commit: only test
26db86b HEAD@{5}: commit (initial): only test

[root@t-luhx01-v-szzb git]# git reset --hard 3fcf6a5
HEAD is now at 3fcf6a5 Modify test.txt
```

**删除文件**

当我们删除文件后，查看状态能发现删除的操作记录

```
[root@t-luhx01-v-szzb git]# git status
On branch master
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	deleted:    test.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

如果确定该文件就是要删除的文件，可以执行rm选项并提交修改

```
[root@t-luhx01-v-szzb git]# git rm test.txt
[root@t-luhx01-v-szzb git]# git commit -m "Deleted text.txt"
```

要是发现是误删除，我们也可以通过checkout撤销操作

```
[root@t-luhx01-v-szzb git]# git checkout -- test.txt
```

从来没有被添加到版本库就被删除的文件，是无法恢复的！

**忽略文件**

在工作区目录通过创建`.gitignore`特殊文件，并将需要忽略的文件名填写进去即可忽略这些文件，如果需要强制将忽略的文件add进去，可以使用`-f`选项，也可以用`git check-ignore -v`检查对应的配置记录

**命令别名**

在日常使用过程中，部分命令太长了记不住，我们可以将其设置一个别名，更方便个人使用。例如`git reset HEAD`撤销暂存区修改，我们可以设置为rollback

```
[root@t-luhx01-v-szzb git]# git config --global alias.rollback 'reset HEAD'
```

将log查看分支信息设置别名为lg

```
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

**远程仓库管理**

关联远程仓库

```
git remote add test git@server-name:path/test.git
```

查看远程仓库

```
git remote -v
```

推送至master主分支

```
git push origin master
```

克隆仓库

```
git clone git@github.com:xxxx/test.git
```

拉取分支，会自动与唯一的跟踪分支合并

```
git pull
```

## 暂存区

Git中存在暂存区的概念，在工作区下有一个隐藏目录`.git`，它是Git的版本库，其中最重要的就是称之为stage的暂存区，还有自动创建的master分支以及指向master的指针HEAD

在我们向Git添加文件时，实际是分为两个步骤，第一个是通过git add把文件修改添加到暂存区，接着通过git commit提交更改的内容，将暂存区所有内容提交到当前分支中

下面我们通过一个测试案例来验证一下，我们先添加一行内容

```
[root@t-luhx01-v-szzb git]# cat test.txt 
this is test file
one line
```

通过add添加到暂存区，然后再进行第二次修改

```
[root@t-luhx01-v-szzb git]# git add test.txt 
[root@t-luhx01-v-szzb git]# cat test.txt 
this is test file
one line
two line
```

commit提交后，查看当前的状态

```
[root@t-luhx01-v-szzb git]# git commit -m "change context"
[master ec0e82a] change context
 1 file changed, 1 insertion(+)
[root@t-luhx01-v-szzb git]# git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   test.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

可以看到我们第二次的修改并没有被提交，正是因为第二次提交没有进入暂存区，而commit是提交暂存区的内容，我们可以查看工作区和版本库中当前版本的区别

```
[root@t-luhx01-v-szzb git]# git diff HEAD -- test.txt 
diff --git a/test.txt b/test.txt
index 51f856e..4c806d2 100644
--- a/test.txt
+++ b/test.txt
@@ -1,2 +1,3 @@
 this is test file
 one line
+two line
```

如果想要提交第二次的修改，需要再次执行add和commit

## 分支管理

**创建和合并分支**

在Git中，Master分支指向最新的提交，再用HEAD指向MASTER，就能确定当前的分支，以及分支的提交点。每次提交，MASTER就会向前推进，随着不断提交，分支线越来越长。当我们创建新的分支之后，将HEAD指向新的分支，即当前分支为新分支，后续所做的操作都是在新的分支线上不断延伸。当需要合并分支时，只需要把master指向新分支当前提交，就完成了合并。

创建并切换分支

```
[root@t-luhx01-v-szzb git]# git checkout -b dev
Switched to a new branch 'dev'
```

查看分支

```
[root@t-luhx01-v-szzb git]# git branch
* dev
  master
```

修改文件

```
[root@t-luhx01-v-szzb git]# echo "two line" >> test.txt 
[root@t-luhx01-v-szzb git]# git add test.txt 
[root@t-luhx01-v-szzb git]# git commit -m "add two line"
[dev 18541da] add two line
 1 file changed, 1 insertion(+)
```

切回master分支

```
[root@t-luhx01-v-szzb git]# git checkout master
Switched to branch 'master'
[root@t-luhx01-v-szzb git]# git branch
  dev
* master
```

查看文件内容，发现并没有之前新增的内容，接下来进行分支合并。git merge能够合并指定分支到当前分支中

```
[root@t-luhx01-v-szzb git]# git merge dev
Updating ec0e82a..18541da
Fast-forward
 test.txt | 1 +
 1 file changed, 1 insertion(+)

[root@t-luhx01-v-szzb git]# cat test.txt 
this is test file
one line
two line
```

在可能的情况下，Git会使用Fast-forward模式，在这种模式下删除分支后，会丢失分支信息，如果强制禁用Fast-forward，Git会在merge时生成一个新的commit，这样从分支历史中就可以看到分支信息。我们可以通过–no-ff强制禁用，因为会生成新的commit，我们合并时添加一下描述

```
git merge --no-ff -m "merge with no-ff" dev
```

删除dev分支

```
[root@t-luhx01-v-szzb git]# git branch -d dev
Deleted branch dev (was 18541da).
[root@t-luhx01-v-szzb git]# git branch
* master
```

**合并冲突**

当两个分支内容存在冲突时，合并会出现错误，必须手动解决冲突问题。例如我们新建一个dev分支，插入一行记录

```
[root@t-luhx01-v-szzb git]# git checkout -b dev
Switched to a new branch 'dev'
[root@t-luhx01-v-szzb git]# git branch
* dev
  master
[root@t-luhx01-v-szzb git]# vi test.txt 
[root@t-luhx01-v-szzb git]# echo "five line" >> test.txt 
[root@t-luhx01-v-szzb git]# git add test.txt \
> 
[root@t-luhx01-v-szzb git]# git commit -m "add five line"
[dev f5de8d1] add three line
 1 file changed, 1 insertion(+)
```

切回master分支再插入一行相同记录

```
[root@t-luhx01-v-szzb git]# git checkout master
Switched to branch 'master'
[root@t-luhx01-v-szzb git]# echo "six line" >> test.txt 
[root@t-luhx01-v-szzb git]# git add test.txt 
[root@t-luhx01-v-szzb git]# git commit -m "add six line"
[master 8e81508] add three line
 1 file changed, 1 insertion(+)
```

合并分区时出现错误

```
[root@t-luhx01-v-szzb git]# git merge dev
Auto-merging test.txt
CONFLICT (content): Merge conflict in test.txt
Automatic merge failed; fix conflicts and then commit the result.
```

此时再看文件中的内容，列出了文件冲突的部分，我们需要手动处理

```
<<<<<<< HEAD
six line
=======
five line
>>>>>>> dev
```

我们可以通过git log查看分支合并图

```
[root@t-luhx01-v-szzb git]# git log --graph --pretty=oneline --abbrev-commit
*   eb47c7c merge
|\  
| * e2dd8ec add five line
* | 527d636 add six line
|/  
*   9b08349 1
```

**临时保存区**

当我们在分支工作到一半的情况下，没办法提交任务时，我们可以通过stash暂时将工作内容保存起来，待后续接着完成。

```
[root@t-luhx01-v-szzb git]# git status
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   test.txt

no changes added to commit (use "git add" and/or "git commit -a")
[root@t-luhx01-v-szzb git]# git stash
Saved working directory and index state WIP on dev: e2dd8ec add five line
HEAD is now at e2dd8ec add five line
[root@t-luhx01-v-szzb git]# git status
On branch dev
nothing to commit, working directory clean
```

可以看到，我们执行stash之后，工作区状态变成干净的了。如果我们要找回之前工作的内容的话，可以对其进行恢复

```
[root@t-luhx01-v-szzb git]# git stash list
stash@{0}: WIP on dev: e2dd8ec add five line
[root@t-luhx01-v-szzb git]# git stash apply stash@{0}
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   test.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

**提交复制**

如果我们想要从其它分支复制一个提交操作到当前分支从可以通过cherry-pick选项完成，指定对应的commit id即可

```
git cherry-pick 18541da
```

## 标签管理

标签能像commit id一样唯一标识发布的各个版本，相比commit id可以进行自定义，更易理解和操作。

创建标签非常简单，切换到对应分支上，执行tag选项即可

```
[root@t-luhx01-v-szzb git]# git tag app-v1.0
[root@t-luhx01-v-szzb git]# git tag
app-v1.0
```

当然，我们也可以针对指定的commit id来打标签

```
[root@t-luhx01-v-szzb git]# git log --pretty=oneline --abbrev-commit
e2dd8ec add five line
9b08349 1
8cd159e add line
2c4226f add line
e292e7d Merge branch 'dev' -m "merge"
d4c71d5 add four line
d94a43a add four line
56c52e2 Merge branch 'dev'
8e81508 add three line
f5de8d1 add three line
18541da add two line
ec0e82a change context
3fcf6a5 Modify test.txt
837262c only test
51de587 only test
26db86b only test
[root@t-luhx01-v-szzb git]# git tag app-v0.5 8e81508
[root@t-luhx01-v-szzb git]# git tag
app-v0.5
app-v1.0
```

查看tag信息

```
[root@t-luhx01-v-szzb git]# git show app-v1.0
commit e2dd8ec4b1944443ffb24ad3dbd8c1f21048b975
Author: luhengxing <690627803@qq.com>
Date:   Fri Aug 7 10:14:58 2020 +0800

    add five line

diff --git a/test.txt b/test.txt
index 70b872e..e767fe3 100644
--- a/test.txt
+++ b/test.txt
@@ -3,3 +3,4 @@ one line
 two line
 three line
 four line
+five line
```

推送本地tag

```
git push origin <tagname>
```

推送所有未推送的tag

```
git push origin --tags
```

删除本地tag

```
git tag -d <tagname>
```

删除远程tag

```
git push origin :refs/tags/<tagname>
```