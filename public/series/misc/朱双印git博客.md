
# git 教程

来源：朱双印博客

无论之前code_test目录中是否包含其他文件，我们都能够使用”git init”命令将code_test目录变为一个git仓库，即使code_test目录中原来就包含其他文件，当我们执行”git init”命令以后，code_test目录中的`原有文件也不会被git管理`，因为如果想要管理这些文件，我们还需要一些其他操作，才能将它们纳入到git的追踪范围以内，也就是说，git init命令只是让code_test目录拥有了版本管理的能力，无论code_test目录中原来是否存在文件，”.git”目录都是新创建出来的，从git仓库的角度来说，这就是一个新的仓库，还没有任何文件被这个仓库所管理。

git config --system：使对应配置针对系统内所有的用户有效
git config --global：使对应配置针对当前系统用户的所有仓库生效
git config --local：使对应配置只针对当前仓库有效
local选项设置的优先级最高。

当我们使用git进行版本管理时，git会将我们的文件和目录结构转化成git方便操作的数据，也就是说，git会将我们的文件和目录转化成一种叫做”`git对象`”的东西，然后再对这些”git对象”进行管理，从而实现版本管理的目的，这些git对象存放在git的对象库中。

我们眼中的文件会被git转化成”块”(`blob`)

我们眼中的目录会被git转化成”树”(`tree`)

我们眼中的状态会被git转化成”提交”(`commit`)

blob、tree、commit都是git对象，是三种不同类型的git对象

一个blob就是由一个文件转换而来，blob对象中只会存储文件的数据，而不会存储文件的元数据。

一个tree就是由一个目录转化而来，tree对象中只会存储一层目录的信息，它只存储它的直接文件和直接子目录的信息，但是子目录中的内容它并不会保存。

一个commit就是一个我们所创建的提交，它指向了一个tree，这个tree保存了某一时刻项目根目录中的直接文件信息和直接目录信息，也就是说，这个tree会指向直接文件的blob对象，并且指向直接子目录的tree对象，子目录的tree对象又指向了子目录中直接文件的blob，以及子目录的直接子目录的tree，依此类推。

上图中的”.git”目录就是所谓的”`版本库`”，而上图中除了”.git”目录以外的其他文件和目录组成了”`工作区`”，换句话说就是，进入git_test目录，排除”.git”目录以后的区域就是工作区，工作区的概念非常容易理解，因为我们的实际工作就发生在这个区域，我们在这个区域创建文件和目录、编辑文件、删除文件和目录，这些操作都是在工作区完成的，而当我们需要把工作区的状态保存到版本库时，则需要借助到”版本库”区域了

“git status”：查看有没有变更的状态，并且查看哪些变更已经加入了暂存区，红色的变更表示只存在于工作区，还未加入暂存区，绿色的变更表示已经加入到暂存区，这些变更将会被提交。

“git add”：将需要进行提交的变更加入到暂存区。

“git commit”：**将所有加入暂存区的变更作为一个变更集合，创建提交**。

“git log”：查看提交历史，此命令有很多实用参数可以使用，利用这些参数可以通过不同的方式查看历史

“git cat-file -t 哈希值”查看对象的`类型`

“git cat-file -p 哈希值”查看对象的`内容`

通过简短的哈希值获取到整个哈希值，如下：
git rev-parse 13614

“`.git/objects/`”目录中存放了git对象，那么之前所描述的”索引”信息，存放在哪里了呢？索引的信息其实存放在” `.git/index`”文件中，我们无法直接查看这个文件内容，如果想要查看这个文件中的索引信息，可以使用如下命令：
git ls-files -s

无论是回退操作，还是使用git log命令查看日志，都会在命令执行后的返回信息中看到一个词，这个词就是”`HEAD` “，那么”HEAD “是什么意思呢？你可以把”HEAD “理解成一个指针，这个指针通常指向了当前”分支”的最新的提交

`git branch test`
上述命令的意思是，根据当前所在分支（master分支）创建一个名为test的分支，但是并不切换到新的分支（test分支），仍然停留在当前分支（master分支）。

注：在没有创建test分支时，我们可以使用”`git checkout –b test`”命令同时完成创建test分支并检出test分支的操作。

## diff

git diff
比较工作区和暂存区
 
git diff HEAD
比较工作区和当前分支最新的提交，你可以把HEAD换成别的分支的名字，比如test分支，"git diff test"表示比较当前工作区和test分支最新的提交之间的差异，也可以把HEAD替换成任何一个commit的ID，表示比较当前工作区和对应提交之间的差异。
 
git diff --cached
比较暂存区和当前分支最新的提交
 
上述命令都是比较所有文件的差异，如果想要指定文件，可以使用"--"指定文件的路径，文件路径可以有多个，用空格隔开。
 
git diff -- file1
git diff -- ./file1
只比较工作区和暂存区中file1文件的差异
 
git diff -- file1 file2
只比较工作区和暂存区中file1以及file2文件的差异
 
git diff -- dir1/d1/f1
只比较工作区和暂存区中dir1/d1/f1文件的差异
 
git diff -- dir1/
只比较工作区和暂存区中dir1目录中所有文件的差异
 
git diff HEAD -- ./file1
只比较工作区和当前分支最新的提交中file1文件的差异，HEAD可以替换成分支名或者commitID
 
git diff testbranch -- ./file1
只比较工作区和testbranch分支最新的提交中file1文件的差异
 
git diff --cached testbranch
比较暂存区和testbranch分支最新的提交
 
git diff --cached testbranch --./file1
只比较暂存区和testbranch分支最新的提交中file1文件的差异
 
git diff HEAD~ HEAD
比较当前分支中最新的两个提交之间的差异
 
git diff HEAD~ HEAD -- file1
比较当前分支中最新的两个提交中的file1文件的差异
 
git diff commitID1 commitID2
比较两个commit之间的差异
git diff commitID1..commitID2
同上，比较两个commit之间的差异，两个命令等效
 
git diff branch1 branch2
比较两个分支上最新提交之间的差异
git diff branch1..branch2
同上，比较两个分支上最新提交之间的差异，两个命令等效

## 后悔药

”git reset HEAD”命令可以帮助我们把暂存区恢复到未暂存任何工作区修改的状态（即与最新的commit的状态保持一致）

我们只是想要从暂存区去除f2的变更，f1的变更仍然保留在暂存区中，那么我们可以执行如下命令：
git reset HEAD -- f2

我们使用了”git reset HEAD”命令将所有已经暂存的变成从暂存区撤销了（即暂存区与最近提交中的状态一致）。

我们使用了”git reset –hard HEAD”命令将所有逻辑区域恢复成了最近的提交中的状态（即工作区、暂存区都与最近提交中的状态一致）。

我们可以用一张表格来总结git reset命令的不同选项影响的区域，如下表所示：

 	工作区	索引	HEAD
–soft	否	否	是
–mixed	否	是	是
–hard	是	是	是

如果你想要的一次性将所有工作区的变更全部撤销，也可以仓库的根目录中执行如下命令
git checkout -- ./*

撤销已经添加到暂存区中的修改，即让暂存区与最近的提交保持一致，可以使用如下命令，如下三条命令等效

git reset
git reset HEAD
git reset --mixed HEAD
也可以针对某些文件撤销暂存区中的修改，命令如下

git reset HEAD -- file1 file2
撤销所有暂存区和工作区中的所有变更

git reset --hard HEAD
回退到指定的提交

git reset --hard commitID
你已经将部分提交暂存到了暂存区，然后继续在工作区工作，工作区产生了新的变更，但是这些新变更没有添加到暂存区，此时你创建了提交，刚刚创建完提交你就后悔了，你想要的回到提交创建前一刻的状态，可以使用如下命令

git reset --soft HEAD~
使用如下命令可以撤销工作区中file1文件的相关变更，可以细分为两种情况

git checkout -- file1
情况一：你先修改了file1，并且暂存了，然后又修改了file1，在工作区产生了新的变更，此时执行上述命令，会将工作区中最新的变更撤销，工作区中的file1将会变成暂存区中file1的状态。

情况二：你修改了file1，暂存区中没有file1相关的变更，此时执行上述命令，会将工作区中最新的变更撤销，工作区中的file1将会变成最近一次提交中file1的状态。

### 撤销或修改过去的commit

修改最近一次 commit 的 message
git commit --amend

修改过去的commit
git rebase -i (commit)

撤销以前的commit再提交
git reset HEAD~3
git add .
git commit -am "feat(user): add user resource"

## git clone 

git clone --bare 表示只克隆.git

## merge

https://www.zsythink.net/archives/3470

现在，我们着重的看一下merge命令以及常用的一些参数。

git merge A
上述命令表示将A分支合并到当前分支。

git merge --no-ff A
上述命令表示将A分支合并到当前分支，但是明确指定不使用”Fast-forward”的模式进行合并， “–no-ff”中的”ff”表示 “Fast-forward”，即使合并条件满足”Fast-forward”模式，也不使用快进的方式进行合并，而是使用创建合并提交的方式进行合并。

git merge --ff-only A
上述命令表示将A分支合并到当前分支，但是只有在符合”Fast-forward”模式的前提下才能合并成功，在不符合”Fast-forward”模式的前提下，合并操作会自动终止，换句话说就是，当能使用”Fast-forward”模式合并时，合并正常执行，当不能使用”Fast-forward”模式合并时，则不进行合并。

git merge --no-edit A
上述命令表示将A分支合并到当前分支，但是没有编辑默认注释的机会，也就是说，在创建合并提交之前不会调用编辑器（上文的示例中会默认调用vim编辑器，以便使用者能够有机会编辑默认的注释信息），换句话说就是，让合并提交直接使用默认生成的注释，默认注释为” Merge branch ‘BranchName’ “

git merge A --no-ff -m "merge A into master,test merge message"
上述命令表示将A分支合并到当前分支，并且使用-m参数指定合并提交对应的注释信息。

git branch -D new # 强制删除

git branch -d new # 非强制

其实，在没有冲突能够正常合并的情况下，我们也可以明确指定不自动创建提交，而是手动的创建提交，我们只需要借助”–no-commit”参数即可，示例如下
git merge --no-commit new

## 其他

git commit -am "注释"
上述命令表示：省略”git add”操作，自动完成暂存并创建提交，换句话说就是，直接将工作区中的所有变更创建成一个提交，注意：完全新建没有被git跟踪过的文件不会自动暂存。

git rm testfile
上述命令表示：删除testfile文件，并自动将变更暂存，效果相当于在文件系统中删除testfile，然后手动暂存。

git mv testfile tf
上述命令表示：将testfile重命名成tf，并自动将变更暂存，效果相当于在文件系统中重命名了testfile，然以手动暂存。

使用 `git switch` 来切换分支，使用 `git restore` 来撤销本地修改

## 远程仓库

我们可以在本地仓库中执行”git push origin master:master”命令，”git push origin master:master”命令的作用是将本地的内容推送到远程origin仓库中，推送的本地分支名是master，远程分支名也是master，没错，”master:master”中冒号左边的master代表你要推送的本地master分支，冒号右边的master代表推送到远程仓库中的master分支

git branch -vv 查看与远程仓库的关系

`git push –u origin new:new` 会在推送本地new分支到远程仓库的同时，直接将本地new分支的上游分支设置为远程仓库中的new分支

git branch -avv 查看远程仓库的分支

`git remote add origin git@github.com:zsythink/test1.git`
你肯定看明白了，你只需要在本地仓库中执行如上命令，即可将本地仓库的远程仓库指定为origin，远程仓库的地址就是”git@github.com:zsythink/test1.git”，远程仓库名和仓库地址你可以根据实际情况进行设定，当然，这个地址也可以是https格式的地址。

以remotes/origin开头的分支代表了origin远程仓库中的分支，这些分支实际存放在你的本地仓库中，但是这些分支代表了远程仓库中的分支

git push --all
此命令表示当本地分支与上游分支同名时，push所有分支的更新到对应的远程分支。

`git fetch`
此命令表示获取远程仓库的更新到本地，但是不会更新本地分支中的代码。

`git pull remote branchA`
此命令表示将remote仓库的A分支pull到本地当前所在分支，如果你想要pull到本地的A分支，需要先checkout到本地A分支中。

git pull remote branchA:branchB
此命令表示将remote仓库的A分支pull到本地的B分支，在成功的将远程A分支pull到本地B分支后（如果远程A到本地B的pull操作失败了，后面的操作不会执行），再将远程A分支pull到本地的当前所在的分支。

`git pull`
此命令表示当本地分支与上游分支同名时，对当前分支执行pull操作，对其他分支执行fetch操作，具体的差异主要取决于对应的远程分支有没有更新。

其实，上述fetch+merge的操作步骤可以通过一条命令完成，这条命令就是”git pull”命令，也就是说，”git pull”命令会完成”git fetch”命令和”git merge”命令两条命令的工作
