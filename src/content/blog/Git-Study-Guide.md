---
title: 学习使用 Git
date: 2023-02-05 13:22:40
category: 笔记
---

Git 属于「分布式版本管理系统」，其特点在于克隆仓库时，会顺带将版本历史一同克隆，所以假设云端的仓库发生了修改，那么在本地的仓库进行同步操作的时候，云端的改动也会应用到本地当中，同时版本历史也记录着这些改动，并且这些改动可以被还原（例如恢复被删除的文件）。

<!-- more -->

「分布式」体现在每一个克隆了仓库的开发者都可以单独对版本进行管理，并且不需要网络就可以 commit 。

## 命令

### clone

```Shell
git clone
```

用于将仓库克隆到本地。

用例：

- `git clone git@github.com:username/repositoryName.git myRepository`：将`username`用户的`repositoryName`仓库克隆到当前目录到`myRepository`文件夹下，其中`myRepository`是一个指定文件夹的操作，可以省略不写，不写的话默认以仓库名作为文件夹名称，然后克隆到这个文件夹中。

### log

```Shell
git log
```

查看 commits 历史，其中新的 commit 始终打印在屏幕上边。一个 commit 记录大概长这样：

> commit sc872s0vvke49ee6a7bac1952a6c5942b1195a11 (HEAD -> main, origin/main, origin/HEAD)
> Author: Aiden Lin <example@yahoo.com>
> Date:   Fri Feb 3 12:00:00 2023 +0800
>
> Initial commit

一个记录大致涵盖了几条信息，首先`11867c9dfce49ee6a7bac1952a6c5942b1195a11`是这个 commit 的哈希值，往下是这次 commit 的提交者和他的邮箱，以及提交日期，最后是这次 commit 的备注。而在哈希值的右边，`HEAD`代表了一个指针，其往往会指向最新一次的 commit 。除了指针，分支也算是引用的一种，例如`main`分支，其可以指向一个 commit ，也可以被`HEAD`指向，实际中往往是后者。

如果仓库中存在多个分支，那么在查看记录的时候可以通过添加参数`--graph`来生成友好的图形化打印，由此可以更加清晰地查看提交情况。

### add

```Shell
git add
```

用例：

- `git add README.md`：将 README.md 添加到暂存区（staging area），在 commit 时会将该文件的改动提交。
- `git add .`：将所有文件添加到缓存区，在 commit 时会将改动提交。

### commit

```Shell
git commit
```

将暂存区的文件的改动进行提交，回车后会出现允许输入文本的区域，此时输入的内容将作为本次 commit 的备注（就是用于说明本次提交做了什么改动，此为可选操作）。

用例：

- `git commit -m '修复了 BUG'`：提交这次的修改，并且为其添加备注：“修复了 BUG”。

### push

```Shell
git push
```

将所有 commits 推送到云端的仓库。

用例：

- `git push origin main`：其中`origin`为云端仓库的默认名称，`main`是需要上传到的分支的名称。

### pull

```Shell
git pull
```

将云端仓库拉取到本地，为了同步别人的改动，或者是保持仓库的最新状态。

用例：

- `git pull origin main`：其中`origin`为云端仓库的名称，`main`是需要拉取的分支的名称。

`git pull`实际上由两步操作组成，分别是`git fetch`和`git merge`，git 首先会执行 fetch 操作将云端仓库的镜像（`origin/main`、`origin/HEAD`等）同步到本地，接着再把这些镜像所指向的 commit 同步到本地，这样 fetch 的操作就完成了。而 merge 的工作则是调整本地的分支和 HEAD 指针的指向，让它们指向和镜像相同的 commit ，这样就能保持本地和云端同步了。

不过还需要注意，对于这条命令`git pull origin main`来说，在执行 merge 的时候，git 只会调整本地的 main 分支，使其与云端镜像保持同步，但是假设云端仓库和本地仓库都存在名为 feature1 的分支，并且云端和本地的 feature1 分支的 commits 并不相同，那么在 pull 的时候，对于 feature1 来说，只会执行 fetch 而不执行 merge ，也就是云端的 feature1 是不会自动和本地的 feature1 merge 的。

### 分支相关

```Shell
git branch
git checkout
git merge
```

创建、切换、合并分支。

用例：

- `git branch feature1`：创建一个名为`feature1`的分支，这个分支默认指向最新的 commit ，创建好的分支可以在`git log`中查看到。
- `git checkout feature1`：切换到`feature1`分支。另外此时在`git log`中可以看到，`HEAD`指针从原来指向`main`分支变成了指向`feature1`分支。
- `git merge feature1`：在执行这个操作前，需要先执行切换分支的操作，以保证当前所在分支不是被合并的分支（不是`feature1`），例如先切换到`main`分支，然后再执行。在合并分支时会出现几种情况，最理想的一种是「Fast-forward」，也就是被合并的分支的开发进度领先于主分支，合并的结果就是主分支的指针直接指向被合并的分支所指向的 commit 就行了。
- `git branch -d feature1`：删除`feature1`分支。

### reset

```Shell
git reset
```

将仓库恢复到某个 commit 的状态。

用例：

- `git reset --hard abcdef`：恢复到 commit 为`abcdef`的状态。

### status

```Shell
git status
```

查看当前分支的改动状态，包括位于暂存区（staging area）的和修改了但是还没有添加到暂存区的。具体来说，假设在`README.md`中默认有这样的内容：

```markdown
# Learn-Git
some text
```

此时执行`git status`会得到如下结果：

> 无文件要提交，干净的工作区

随后对文件进行修改：

```markdown
# Learn-Git
some text

More text
```

此时执行`git status`会得到如下结果：

> 尚未暂存以备提交的变更：
> （使用 "git add <文件>..." 更新要提交的内容）
> （使用 "git restore <文件>..." 丢弃工作区的改动）
> 修改：     README.md
>
> 修改尚未加入提交（使用 "git add" 和/或 "git commit -a")

可以看到 git 已经探测了改动，并且`修改：     README.md`这一行文字将以 **红色** 字体展示，此时执行`git add README.md`将改动提交到暂存区，然后再执行`git status`会得到如下结果：

> 要提交的变更：
>（使用 "git restore --staged <文件>..." 以取消暂存）
> 修改：     README.md

此时`修改：     README.md`将以 **绿色** 字体展示。此时再次修改文件：

```markdown
# Learn-Git
some text

More text

some code
```

再执行`git status`会得到如下结果：

> 要提交的变更：
> （使用 "git restore --staged <文件>..." 以取消暂存）
> 修改：     README.md
>
> 尚未暂存以备提交的变更：
> （使用 "git add <文件>..." 更新要提交的内容）
> （使用 "git restore <文件>..." 丢弃工作区的改动）
> 修改：     README.md

上面的`修改：     README.md`以 **绿色** 字体展示，下面的则以 **红色** 字体展示。由此得知，git 可以探测到文件的改动，并且未被添加到暂存区的改动，git 将以 **红色** 字体进行提示，而提交到暂存区但是没有进行 commit 的改动，git 将以 **绿色** 字体进行提示。进一步说，`git add`针对的是改动而不是文件，每一次「add」实际上是将该文件的这一次改动添加到暂存区，而不是说将整个文件添加到暂存区。

## Feature Branching

所谓「Feature Branching」即是在不影响主分支代码的情况下进行开发，也就是为需要开发的新功能创建一条新分支，然后在这条分支上编写代码，这样一来，无论开发完成与否，都不会影响到主分支的代码。

### 本地合并

当功能开发完毕，需要合并代码时，一般有两种做法，第一种就是在本地合并，然后推向云端仓库。具体来说，就是先切换到主分支，然后`git pull`……对，不是立刻`git merge`，而是先 pull 一下云端仓库，以此保证主分支的同步，然后再在本地 merge 分支，最后再推向云端仓库。这么做没什么大问题，主要是在 push 之前有可能会发生主分支又被别人以迅雷不及掩耳之势修改从而导致不能「Fast-forward」也就是会导致 push 失败……如果真的是这样，那就只能再 pull 一次，然后再 push ，如此反复。

### 基于 GitHub

第二种做法则是基于 GitHub 中的「pull request」来实现，这种做法就不需要在本地 merge ，而是直接把分支推送到云端仓库。例如现在有一个分支名为「feature1」，那么在开发完成后，就直接`git push origin feature1`，此时在仓库的网页页面顶部就会出现一个提示：![feature1 已被推送](/images/note/feature1-had-recent-pushes.png)

而在提示右边的「Compare & pull request」按钮将会在云端合并分支时用到，点击按钮就会进入到一个「Open a pull request」的页面，在这个页面中可以写一些关于本次更新的说明，然后点击右下角的「Create pull request」按钮来创建一个 pull request 。pull request 在创建好以后，可以在右边的边栏中添加「Reviewers」，让他们对本次更新进行检查。此外还可以添加「Labels」，以确定本次更新的主题。

pull request 创建好以后，Reviewers 可能会提出一些建议，基于这些建议，需要在本地对代码进行修改，修改完成后，只需要再次`git push origin feature1`，改动就会被推送到 pull request 中，Reviewers 可以立刻看到这些 commits 。如果没什么问题了，就可以点击下方的「Merge pull request」按钮来合并分支，然而这是个「drop down menu」，点击后提供了 3 个选项：

- `Create a merge commit`（All commits from this branch will be added to the base branch via a merge commit.）：这是默认选项，会将该分支下所有的 commits 合并到主分支中。需要注意的是，GitHub 在合并的时候添加了`--no-ff`参数，该参数表示不使用「Fast-forward」特性，所以在合并后，会在顶部额外产生一个新的 commit ，这个 commit 的标题和描述就是在按下「Merge pull request」按钮后填写的内容。

![填写这次 merge commit 的信息](/images/note/merge-description.png)

![Create a merge commit 示意图](https://docs.github.com/assets/cb-5407/images/help/pull_requests/standard-merge-commit-diagram.png)

- `Squash and merge`（The x commits from this branch will be combined into one commit in the base branch.）：将分支上的所有 commits 压缩成一个 commit 然后 merge 到主分支中，merge 采用「Fast-forward」选项。

![Squash and merge 示意图](https://docs.github.com/assets/cb-5742/images/help/pull_requests/commit-squashing-diagram.png)

- `Rebase and merge`（The x commits from this branch will be rebased and added to the base branch.）：将分支上的 commits 逐一添加到主分支上，并且不会额外生成 commit 。

最后，在云端仓库完成了合并以后，本地也需要合并，只不过是使用`git pull`来合并，而不是`git merge`了。
