## 创建版本库

只需要执行命令`git init`命令，就可以把当前的目录变为GIt可以管理的仓库

```powershell
$ git init
Initialized empty Git repository in E:/LearnGit/.git/
```

这时候当前目录会多了一个文件夹`.git`，注意这是一个隐藏的目录。

### 把文件添加到版本库

把文件添加到Git仓库需要两步：

1. 使用命令`git add`告诉Git，把文件添加到仓库

```powershell
$ git add readme.txt
warning: LF will be replaced by CRLF in readme.txt.
The file will have its original line endings in your working directory
```

2. 使用命令`git commit`告诉Git，把文件提交到仓库

```powershell
$ git commit -m "wrote a readme file"
[master (root-commit) 705e5a5] wrote a readme file
 Committer: unknown <xxxx@xxxx.internal>
Your name and email address were configured automatically based
on your username and hostname. Please check that they are accurate.
You can suppress this message by setting them explicitly. Run the
following command and follow the instructions in your editor to edit
your configuration file:

    git config --global --edit

After doing this, you may fix the identity used for this commit with:

    git commit --amend --reset-author

 1 file changed, 3 insertions(+)
 create mode 100644 readme.txt
```

注意我这里还没有配置Git账户相关的信息，可以先按照它的提示去进行更改。

注意`git commit`命令后的`-m` 参数，表示这次提交相应的说明信息，

> Git添加文件需要`add`和`commit`两步，因为`add`只是放在一个暂存库中，而`commit`则是把多次`add`的文件进行提交。

## 版本控制

当我们修改文件内容后，可以使用命令`git status`查看此时仓库的状态：

```powershell
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

此时提示我们修改了readme.txt文件，但是还没有提交所做的修改。当然我们也可以使用命令`git diff`查看我们修改的具体内容：

```powershell
$ git diff
warning: LF will be replaced by CRLF in readme.txt.
The file will have its original line endings in your working directory
diff --git a/readme.txt b/readme.txt
index ff918b1..b0aa046 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1,3 +1,2 @@   # 总述，减去了第一行，第三行；加上了第一行第二行
-Git is a version control system.   # 原来的句子，用-标识
+Git is a distributed version control system.  # 现在的句子，用+标识
 Git is a free software.`
-   # 原来我这有个空行
    # 现在变成了什么都没有了
```

查看完所做的修改后，就可以进行提交了。首先使用`git add`命令将其提交到暂存库中，在进行`commit`之前，我们再使用命令`git status`看一下此时Git的状态，提示我们将要提交修改后的readme.txt：

```powershell
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   readme.txt
```

然后再使用`git commit`进行真正的提交，提交后再次查看仓库状态：

```powershell
$ git status
On branch master
nothing to commit, working tree clean
```

此时工作目录是干净的，没有需要提交的修改。

> 使用`git status`随时查看仓库的状态，使用`git differ`可以查看修改的具体内容。

### 版本回退

其实每一次的`commit`就相当于我们保存了原仓库的一个快照，**所以恢复的时候也是以`commit`为单位进行回退的**。当我们要查看所有的修改记录时，**可以使用命令`git log`，它会显示我们的commit记录**，且时间上从近到远。

```powershell
$ git log
commit 2a50cf57264a14665ea473bc3c48a61fa23a3e17 (HEAD -> master)
Author: reachwang <wangfuhao0000@126.com>
Date:   Tue Jun 30 10:26:56 2020 +0800

    append GPL

commit ed87a2a6a7e5fc802ddc786171f4874e0ae55992
Author: reachwang <wangfuhao0000@126.com>
Date:   Tue Jun 30 10:23:47 2020 +0800

    add distributed

commit 5ef42564e346d137d91e46627f272986baca21ca
Author: unknown <gzs15721@yajuenet.internal>
Date:   Tue Jun 30 10:02:48 2020 +0800

    wrote a readme file
```

注意能看到在commit后面有一大串字符，它们是`commit id`（版本号），是一个SHA1计算出来的数字。

现在要将文件回退到上一个`commit`版本。首先Git中，使用`HEAD`来表示当前版本，也就是最新提交的`2a50cf...`版本，那么上一个版本就用`HEAD^`表示，上上个版本就用`HEAD^^`，当然要回退的版本较多时，就是用类似`HEAD~100`这样的表示。

要把当前版本回退到上一个版本，就可以使用`git reset`命令：

```powershell
$ git reset --hard HEAD^
HEAD is now at ed87a2a add distributed
```

回退完成后，我们再用`git log`查看现在版本库的状态：

```powershell
$ git log
commit ed87a2a6a7e5fc802ddc786171f4874e0ae55992 (HEAD -> master)
Author: reachwang <wangfuhao0000@126.com>
Date:   Tue Jun 30 10:23:47 2020 +0800

    add distributed

commit 5ef42564e346d137d91e46627f272986baca21ca
Author: unknown <gzs15721@yajuenet.internal>
Date:   Tue Jun 30 10:02:48 2020 +0800

    wrote a readme file
```

能看到刚才最新的`append GPL`版本已经看不到了。此时如果我们想前进到最新的版本，就需要找到对应`commit`的`commit_id`，然后再使用命令`git reset`来恢复到指定的版本。**Git提供了一个命令`git reflog`来记录了所有的操作**，从这里就能看到未来的`commit_id`。当然写版本号的时候没必要写全部的，只需要写前几位能唯一的标识它即可。

```python
$ git reset --hard  2a50cf57
HEAD is now at 2a50cf5 append GPL
```

Git的版本回退速度很快，因为Git内部有个指向当前版本的`HEAD`指针，当回退版本时，Git仅仅是把HEAD从一个版本指向了另一个版本，然后再更新**工作区**内的文件（暂存区和工作区后面说）。

>- `HEAD`指向的版本就是当前版本，因此我们可以使用命令`git reset --hard commit_id`来从不同的历史版本之间穿梭
>- 穿梭前，可以使用`git log`命令查看每个版本的`commit_id`，然后再去决定**回退**到哪个版本
>- 要重返未来，用**`git reflog`**查看命令历史，再决定回到未来的哪个版本。

### 工作区和暂存区

工作区就是电脑里面能看到的目录，或者说你的文件夹目录。

**版本库**

工作区有一个隐藏目录`.git`，它是Git的版本库，里面存了很多东西，最重要的就是称为stage的**暂存区**，还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`。

![image-20200630112837586](E:%5C%E7%AC%94%E8%AE%B0%5C%E5%9B%BE%E7%89%87%5Cimage-20200630112837586.png)

前面说提交文件需要两步，其中`git add`就是把文件修改添加到暂存区；`git commit`则是把暂存区的所有内容提交到当前分支。因为我们创建Git版本库时，Git自动创建了唯一一个`master`分支，所以现在`git commit`是往`master`分支上提交更改。

### 管理修改

Git追踪并管理的是修改，而非文件，修改即你新增了一行或者删除了一行或者创建了一个新文件等。我们必须把所有的修改执行`git add`命令放入暂存区后，再进行`git commit`时才会真正执行修改；否则修改文件后直接执行`git commit`后不会执行对应的修改。

### 撤销修改

命令`git checkout --readme.txt`可以把`readme.txt`文件在**工作区**的修改全部撤销，这里有两种情况：

- 一种是`readme.txt`自然修改后还没有被放到暂存区，现在撤销修改就回到和版本库一模一样的状态
- 一种是`readme.txt`已经添加到暂存区后，又做了修改，现在撤销修改就回到添加到暂存区后的状态

总之就是让这个文件回到最近一次`git commit`或`git add`时的状态。

`git checkout -- file`命令中的` --`很重要，没有它就变成了“切换到另一个分支”的命令。

当将修改添加到了暂存区但是还没有`commit`时，可以使用命令`git reset HEAD <file>`把暂存区的修改撤销掉，重新放回工作区。

```powershell
$ git reset HEAD readme.txt
Unstaged changes after reset:
M       readme.txt
```

`git reset`既可以回退版本，**也可以把暂存区的修改回退到工作区**。当我们使用`HEAD`时表示最新的版本。

再使用`git status`查看一下，现在暂存区时干净的，工作区有修改：

```powershell
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

然后再使用`git checkout -- readme.txt`来丢弃工作区的修改，这样就完成了全部的回退。

> - 当不小心改了工作区的某个文件内容时，想要丢弃工作区的修改则使用命令`git checkout -- file`
> - 当改完工作区内容后还添加到了暂存区，想丢弃修改分两步：第一步使用`git reset HEAD <file>`回到了上面的情况，然后再使用上面的命令进行操作
> - 已经提交了不合适的修改到版本库，想撤销本次提交则参考版本回退部分，前提是没有推送到远程库。

### 删除文件

GIt中删除也是一个修改操作，首先添加一个新文件`test.txt`并进行提交：

```powershell
$ git add test.txt

$ git commit -m "add test.txt"
[master 080ba76] add test.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test.txt
```



## 远程仓库

在使用Github之前需要进行一些设置：

1. 设置SSH key。先看用户主目录下有没有.ssh目录，且内部有没有`id_rsa`和`id_rsa.pub`这两个文件。如果没有则打开Shell（Windows下时Git Bash），创建SSH key：

```bash
ssh-keygen -t rsa -C "your_email@example.com"
```

然后连续按回车后，就可以在用户主目录中找到`.ssh`目录，里面有`id_rsa`和`id_rsa.pub`两个文件，这两个就是SSH Key的秘钥对，**`id_rsa`是私钥，不能泄露出去，`id_rsa.pub`是公钥，可以放心地告诉任何人**。

2. 登录GitHub，打开“Account settings”，“SSH Keys”页面，然后点“Add SSH Key”，填上任意Title，在Key文本框里粘贴`id_rsa.pub`文件的内容后，点“Add Key”，就可以看到已经添加的Key。这样在我们向github提交文件的时候，就能确保确实是我们自己提交的。

### 添加远程库



### 从远程库克隆

克隆比较简单，只需要得到项目的git地址后，在目录下执行`git clone`命令即可。

## 分支管理

### 创建与合并分支

在[版本回退](https://www.liaoxuefeng.com/wiki/896043488029600/897013573512192)里，你已经知道，每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。截止到目前，只有一条时间线，在Git里，这个分支叫主分支，即`master`分支。`HEAD`严格来说不是指向提交，而是指向`master`，`master`才是指向提交的，所以，`HEAD`指向的就是当前分支。

一开始的时候，`master`分支是一条线，Git用`master`指向最新的提交，再用`HEAD`指向`master`，就能确定当前分支，以及当前分支的提交点：

![git-br-initial](https://www.liaoxuefeng.com/files/attachments/919022325462368/0)

每次提交，`master`分支都会向前移动一步，这样，随着你不断提交，`master`分支的线也越来越长。

当我们创建新的分支，例如`dev`时，Git新建了一个指针叫`dev`，指向`master`相同的提交，再把`HEAD`指向`dev`，就表示当前分支在`dev`上：

![git-br-create](https://www.liaoxuefeng.com/files/attachments/919022363210080/l)

所以Git创建一个分支很快，因为除了增加一个`dev`指针，改改`HEAD`的指向，工作区的文件都没有任何变化！

不过，从现在开始，对工作区的修改和提交就是针对`dev`分支了，比如新提交一次后，`dev`指针往前移动一步，而`master`指针不变：

![git-br-dev-fd](https://www.liaoxuefeng.com/files/attachments/919022387118368/l)

假如我们在`dev`上的工作完成了，就可以把`dev`合并到`master`上。Git怎么合并呢？最简单的方法，就是直接把`master`指向`dev`的当前提交，就完成了合并：

![git-br-ff-merge](https://www.liaoxuefeng.com/files/attachments/919022412005504/0)

所以Git合并分支也很快！就改改指针，工作区内容也不变！

合并完分支后，甚至可以删除`dev`分支。删除`dev`分支就是把`dev`指针给删掉，删掉后，我们就剩下了一条`master`分支：

![git-br-rm](https://www.liaoxuefeng.com/files/attachments/919022479428512/0)

## 问题

### Git HEAD detached from XXX (git HEAD 游离) 解决办法

[博客链接](https://blog.csdn.net/u011240877/article/details/76273335)

这个问题的原因就是在于我们使用`git checkout <commit_id>`切换到了某次提交，这样HEAD没有指向任何一个分支，就会处于游离状态。

![这里写图片描述](https://img-blog.csdn.net/20170728173951460?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTI0MDg3Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

HEAD 处于游离状态时，我们可以很方便地在历史版本之间互相切换，比如需要回到某次提交，直接 checkout 对应的 commit id 或者 tag 名即可。

它的弊端就是：**在这个基础上的提交会新开一个匿名分支！**

![这里写图片描述](https://img-blog.csdn.net/20170728174105093?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTI0MDg3Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

也就是说我们的提交是无法可见保存的，一旦切到别的分支，游离状态以后的提交就不可追溯了。

![这里写图片描述](https://img-blog.csdn.net/20170728174044548?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTI0MDg3Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

解决办法就是新建一个分支保存游离状态后的提交：

![这里写图片描述](https://img-blog.csdn.net/20170728174140630?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTI0MDg3Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

具体解决操作
git branch -v 查看当前领先多少

4449a91 指向的是 dev1 的最后一次提交
新建一个 temp 分支，把当前提交的代码放到整个分支

checkout 出要回到的那个分支，这里是 dev1

然后 merge 刚才创建的临时分支，把那些代码拿回来

git status 查看下合并结果，有冲突就解决

合并 OK 后就提交到远端

删除刚才创建的临时分支


查看 Log，当前 HEAD 指向本地 dev1 ，和远端 dev1 一致，OK 了！
