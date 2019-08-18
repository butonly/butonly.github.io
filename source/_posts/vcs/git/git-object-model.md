---
title: Git对象模型：一步一步分析Git底层对象模型
date: 2019-06-24T11:00:00.000Z
updated: 2019-08-18T13:47:16.812Z
tags: [vcs,git]
categories: [git]
---



Git是一个非常强大的版本管理工具，有非常多的概念和命令，也有非常多且复杂的用法。但是其底层模型相对而言却非常简单，了解底层对象模型将非常有助于理解上层命令到底对仓库做了什么。如果想要精通Git，了解底层对象模型也是必不可少的。本文将会通过基本的命令操作，分析每一步操作对仓库数据做了哪些改动，进而分析出Git底层对象模型。

## Git基本概念

### SHA

所有用来表示项目历史信息的文件，都是通过一个40个字符的（40-digit）“对象名”来索引的，对象名看起来像这样:

> 6ff87c4664981e4397625791c8ea3bbb5f2279a3

在后边使用对象名时，只使用前面几个字符即可，但是最少需要4个。

你会在Git里到处看到这种“40个字符”字符串。每一个“对象名”都是对“对象”内容做SHA1哈希计算得来的。这样就意味着两个不同内容的对象几乎不可能（理论上是可能发生碰撞的）有相同的“对象名”。

这样做会有几个好处：

* 只要比较对象名，就可以很快的判断两个对象是否相同。

  因为在每个仓库（repository）的“对象名”的计算方法都完全一样，如果同样的内容存在两个不同的仓库中，就会存在相同的“对象名”下。

* 还可以通过检查对象内容的SHA1的哈希值和“对象名”是否相同，来判断对象内容是否正确。

### 对象

每个对象(object) 包括三个部分：类型、大小和内容。大小就是指内容的大小，内容取决于对象的类型。

有四种类型的对象：`"blob"`、`"tree"`、`"commit"` 和 `"tag"`。

几乎所有的Git功能都是使用这四个简单的对象类型来完成的。它就像是在你本机的文件系统之上构建一个小的文件系统。

#### blob (文件) 对象

`blob` 用来存储文件数据，通常是一个文件。

![blob](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/object-blob.png)

#### tree (目录) 对象

`tree` 像一个目录，管理 `tree（子目录）` 或 `blob（文件）`

![tree](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/object-tree.png)

#### commit (提交) 对象

一个 `commit` 只指向一个 `tree`，它用来标记项目某一个特定时间点的状态。`commit` 保存了树根的 `对象名`。

它包括一些关于时间点的元数据，如 `时间戳`、`最近一次提交的作者`、`指向上次提交（commits）的指针` 等等。

![tree](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/object-commit.png)

```
$ git cat-file -p 830f4857e9f579818c5e69104d3e2cc30f1f0d0d
tree f25061fa4f8b3bffbf8ebcc3ab2351efdad2f605
parent 06e13701ac86eb09c2035329a5e1c18f95898cf2
author liuyanjie <x@gmail.com> 1526828841 +0800
committer liuyanjie <x@gmail.com> 1526828841 +0800

commit message
```

#### tag (标签) 对象

![tag](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/object-tag.png)

一个 `tag` 是来标记某一个 `commit` 的方法。

实际上 `tag` 本身是文件名，内容是 `commit` 的对象名。`tag` 是 `commit` 的别名，类似于域名和ip地址的关系。

```
$ cat .git/refs/tags/v0.0.0
830f4857e9f579818c5e69104d3e2cc30f1f0d0d
```

#### commit -> tree -> blob

![object-c-t-b](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/object-c-t-b.png)

从图上可以看出：一个 `commit` 指向了一棵由 `tree` 和 `blob` 构成的 Git 对象树。

### 与其他版本控制系统的区别

Git与你熟悉的大部分版本控制系统的差别是很大的。

也许你熟悉 `Subversion`、`CVS`、`Perforce`、`Mercurial` 等等，他们使用 “增量文件系统” （Delta Storage systems）, 就是说它们存储每次提交(commit)之间的差异。

Git正好与之相反，它会把你的每次提交的文件的全部内容（snapshot）都会记录下来。

这会是在使用Git时的一个很重要的理念。

综上，Git对象模型非常简单，与普通的文件系统非常的相似。


## 一步一步了解Git在做什么

### 初始化仓库

初始化目录并创建一个Git仓库

```sh
$ cd git-obj-model

$ git init
Initialized empty Git repository in /Users/liuyanjie/git-obj-model/.git/

```

看一下初始化的git仓库中到底有些什么内容。

```sh
$ tree .git
.git
├── HEAD
├── config
├── description
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   ├── prepare-commit-msg.sample
│   └── update.sample
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags

8 directories, 15 files
```

```sh
$ cat .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
```

```sh
$ cat .git/HEAD
ref: refs/heads/master
```

查看仓库状态

```sh
$ git status
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)

```

后面忽略 `.git/hooks/` 下的文件。

### 增加一个文件README.md

现在创建一个 `README.md`，查看 `.git` 目录，发现没有什么变化。

```sh
$ echo "# Readme" > README.md

$ tree .git
.git
├── HEAD
├── config
├── description
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags

8 directories, 15 files
```

查看当前状态，可以看到，此时有一个未追踪的 `README.md` 文件，此文件存在于 `工作区` 中。

```sh
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	README.md

nothing added to commit but untracked files present (use "git add" to track)

```

执行 `git add` 命令后，可以看到增加了两个文件：

* .git/index
* .git/objects/f3/954314c1026028e77ea3a765aadefa67b45195

> git add 把文件暂存到索引中为下一次提交做准备，git commit 创建新的提交。

应该知道，这应该是一个 `blob` 类型的文件，里面存储文件内容。

```sh
$ git add README.md

$ tree .git
.git
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── objects
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags

9 directories, 17 files

```

查看当前状态

```sh
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

	new file:   README.md

```

查看对象内容

```sh
$ git cat-file -p f39543
# Readme

```

查看暂存区

```sh
$ git ls-files --stage
100644 f3954314c1026028e77ea3a765aadefa67b45195 0	README.md

```

.git/index 是 Git索引文件，是一个在 `工作区` 和 `仓库` 间的 `暂存区域(staging area)`。

索引是一个二进制格式的文件，里面存放了与当前暂存内容相关的信息，包括暂存的`文件名`、`文件内容的SHA1哈希串值` 和 `文件访问权限`，整个索引文件的内容以暂存的文件名进行排git ls-files --stage序保存的。

因为这个文件记录了将要提交的文件, 所以我们才能够多次修改一起提交(commit)。

所以创建了一个新的提交(commit)，提交的一般是暂存区里的内容, 而不是工作目录中的内容。


一个Git项目中文件的状态大概分成下面的两大类，而第二大类又分为三小类：

1. 未被跟踪的文件（Untracked files）
2. 已被跟踪的文件（Tracked files）
    1. 被修改 未暂存 的文件（Changed but not updated 或 Modified）
    2. 被修改 已暂存 可以 被提交 的文件（Changes to be committed 或 Staged）
    3. 未修改 的文件（自上次提交以来）(Clean 或 Unmodified)

```sh
Changes to be committed:
  (no files)
Changes not staged for commit:
  (no files)
Untracked files:
? .gitignore
```

执行 `git commit` 命令，发现路径下又增加了两个文件：

* .git/objects/9d/978d59f2f22062c0382c859f4c3ef929026303
* .git/objects/c1/067ba0a7ba51f937518c9bc051ea744ca748fe

从上面的结构图，可以想到：

提交可定会产生一个提交对象，提交对象指向一个树对象，树对象包含上一步添加的`blob`对象。

下面将通过查看文件内容验证这一点。

```sh
$ git commit -m "First Commmit" README.md

[master (root-commit) 033fa1a] First Commmit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md

$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5 +
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe +
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags

14 directories, 23 files

```

查看 `033f` 对象内容，此时不知道这个文件类型及内容是什么

此时，`commit` 对象为初始提交，所以并无 `parent` 引用。

```sh
$ git cat-file -p 033f
tree c1067ba0a7ba51f937518c9bc051ea744ca748fe
author liuyanjie <x@gmail.com> 1527521894 +0800
committer liuyanjie <x@gmail.com> 1527521894 +0800

First Commmit

```

查看 `c106` 对象内容，通过上一步，已知此对象是一个 `tree` 类型的对象，通过内容可以看到，树类型的对象实际就是存储每行一条数据的列表。

```sh
$ git cat-file -p c106
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md

```

再看 `.git/refs/heads/master` 文件内容

文件内容只有一行，内容是 `033f` 对象。`master` 即是 `master` 分支的物理表示。

```sh
$ cat .git/refs/heads/master
033fa1ab71f0d54f348f07a3a0ffcefd52804df5

```

同时，还可看到，新增了 `.git/logs` 目录及内容

该目录下保存了各个分支的提交记录

```sh
$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit

$ cat .git/logs/HEAD
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit

```

Tag

```sh
$ git tag v0.0.1 -m 'tag v0.0.1'

```

查看目录文件变化

```sh
$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe
│   ├── cc
│   │   └── f88a6f1649213499841c33f9bb36d1d8756fb7
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags
        └── v0.0.1

15 directories, 25 files

```

查看 `.git/refs/tags/v0.0.1` 文件内容，可以看到内容恰好是新增的 objects `对象名`，所以可想而知 `ccf8` 是一个 `tag` 类型的对象

```sh
$ cat .git/refs/tags/v0.0.1
ccf88a6f1649213499841c33f9bb36d1d8756fb7

```

查看新增的 `ccf8` 文件内容，文件指向了 `commit` 对象 `033f`

```
$ git cat-file -p ccf8
object 033fa1ab71f0d54f348f07a3a0ffcefd52804df5
type commit
tag v0.0.1
tagger liuyanjie <x@gmail.com> 1527523284 +0800

tag v0.0.1

```

通过以上查看 `.git` 目录的变化过程，可以大致分析出Git是如何存储这些文件内容的。

上面只执行了 `git add` 和 `git commit` 两条操作。

### 继续添加文件

```sh
$ echo "# CHANGELOG" > CHANGELOG.md

$ echo "# CONTRIBUTING" > CONTRIBUTING.md

$ git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	CHANGELOG.md
	CONTRIBUTING.md

nothing added to commit but untracked files present (use "git add" to track)

$ git add CHANGELOG.md CONTRIBUTING.md

$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   CHANGELOG.md
	new file:   CONTRIBUTING.md

```

```sh
$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5
│   ├── a0
│   │   └── cf709bc0991b5340080f944d02894dc1596d46
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe
│   ├── c6
│   │   └── b9e95b39b8cd8ead8bbf4b118104741017de1b
│   ├── cc
│   │   └── f88a6f1649213499841c33f9bb36d1d8756fb7
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags
        └── v0.0.1

17 directories, 27 files

```

```sh
$ git ls-files --stage
100644 a0cf709bc0991b5340080f944d02894dc1596d46 0	CHANGELOG.md
100644 c6b9e95b39b8cd8ead8bbf4b118104741017de1b 0	CONTRIBUTING.md
100644 f3954314c1026028e77ea3a765aadefa67b45195 0	README.md

```

分别查看一下各个文件的内容

```sh
$ git cat-file -p a0cf
# CHANGELOG

$ git cat-file -p c6b9
# CONTRIBUTING

$ git cat-file -p f395
# Readme

```

再查看一下树对象的内容，然而并没有任何变化

```sh
$ git cat-file -p c106
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md

```

下面提交这两个文件，可以通过日志方便的查看信息。

```sh
$ git commit --all -m "add CHANGELOG.md and CONTRIBUTING.md files"
[master 28baf4f] add CHANGELOG.md and CONTRIBUTING.md files
 2 files changed, 2 insertions(+)
 create mode 100644 CHANGELOG.md
 create mode 100644 CONTRIBUTING.md

```

同样再看一下`.git`目录下的内容

```sh
$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5
│   ├── 28
│   │   └── baf4f77fb49abf99c18bc1c12363d898f3ced7
│   ├── 2c
│   │   └── affd90cd736e58f516e3988e3af84f5fa42b4f
│   ├── a0
│   │   └── cf709bc0991b5340080f944d02894dc1596d46
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe
│   ├── c6
│   │   └── b9e95b39b8cd8ead8bbf4b118104741017de1b
│   ├── cc
│   │   └── f88a6f1649213499841c33f9bb36d1d8756fb7
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags
        └── v0.0.1

19 directories, 29 files
```

查看一下master上的提交日志，刚刚提交内容在新的一行，与首次提交稍微有点差别，首次提交是commit (initial)。

还可以发现第二次提交的第一列和第一次提交的第二列一样，可以猜到，第一列指向上一次的提交对象。

```sh
$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit
033fa1ab71f0d54f348f07a3a0ffcefd52804df5 28baf4f77fb49abf99c18bc1c12363d898f3ced7 liuyanjie <x@gmail.com> 1527524452 +0800	commit: add CHANGELOG.md and CONTRIBUTING.md files

```

查看 `28ba`

```sh
$ git cat-file -p 28ba
tree 2caffd90cd736e58f516e3988e3af84f5fa42b4f
parent 033fa1ab71f0d54f348f07a3a0ffcefd52804df5
author liuyanjie <x@gmail.com> 1527524452 +0800
committer liuyanjie <x@gmail.com> 1527524452 +0800

add CHANGELOG.md and CONTRIBUTING.md files

```

相比 `033f`，`28ba` 多了 parent 字段 且 parent 字段值是 `033f`

同时 `master` 也指向了新的提交对象。

第一次产生的提交对象同样存在于目录当中。

```sh
$ cat .git/refs/heads/master
28baf4f77fb49abf99c18bc1c12363d898f3ced7

```

在看树对象`2caf`，发现相比之前，多了两行，分别指向新增加的文件。

`README.md` 文件出现在了 `2caf` 对象中，实际上，它同时还存在于 `c106` 对象中。

```sh
$ git cat-file -p 2caf
100644 blob a0cf709bc0991b5340080f944d02894dc1596d46	CHANGELOG.md
100644 blob c6b9e95b39b8cd8ead8bbf4b118104741017de1b	CONTRIBUTING.md
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md
```

```sh
$ git cat-file -p c106
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md

```

到现在为止，整个目录下有3个文件，而 `.git` 目录下已经有多个文件，为了跟踪记录版本。

```sh
$ ls
CHANGELOG.md CONTRIBUTING.md README.md

```

通过以上 `.git` 目录变化，可以发现：

每次提交都会产出一 `commit` 对象，这些 `commit` 通过 `parent`，形成一个由高版本到低版本的链表，追溯这个链表，可以回溯到任意版本。

`commit` 对象保存一个 `tree` 的根节点，根节点下面再包含 `blob` 或 `tree`，类似普通文件系统结构，和上面的图一致，从根节点开始，可以找到某一般版本下的所有文件。

在每一颗树下，因为都是使用类似指针的结构，所以每次修改都是将变化的文件，重新创建一个blob文件，并修改相应指针。

目录中的 `.git/refs/heads/master` 指向某一次提交，当由另外一个分支的时候，会有 `.git/refs/heads/branch-xxx` 文件指向另外一次提交，而初始时与父分支指向相同。


### 再添加一个带有目录文件

```sh
$ mkdir lib

$ echo "// Author: liuyanjie" > ./lib/index.js

$ git add lib/index.js

$ git commit -m "add lib/index.js" lib/index.js
[master 2b9fc85] add lib/index.js
 1 file changed, 1 insertion(+)
 create mode 100644 lib/index.js

```

```sh
$ git cat-file -p 2b9fc85
tree 9609811e44367d44f2915435f4454716e1e535fd
parent 28baf4f77fb49abf99c18bc1c12363d898f3ced7
author liuyanjie <x@gmail.com> 1527556745 +0800
committer liuyanjie <x@gmail.com> 1527556745 +0800

add lib/index.js

```

```sh
$ git cat-file -p 9609
100644 blob a0cf709bc0991b5340080f944d02894dc1596d46	CHANGELOG.md
100644 blob c6b9e95b39b8cd8ead8bbf4b118104741017de1b	CONTRIBUTING.md
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md
040000 tree 2fb9045bb558889ea2bd8cc5d8fe45e7247706da	lib
```

```sh
$ git cat-file -p 2fb9
100644 blob 39a204af28de9b4f0411735e597e0da7416ca35a	index.js

```

上面连续几步和之前的效果一样，但是可以看到，在树对象 `9609` 中，包含了另一个树对象 `2fb9` ，这个对象的内容指向`index.js`文件。

```sh
$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit
033fa1ab71f0d54f348f07a3a0ffcefd52804df5 28baf4f77fb49abf99c18bc1c12363d898f3ced7 liuyanjie <x@gmail.com> 1527524452 +0800	commit: add CHANGELOG.md and CONTRIBUTING.md files
28baf4f77fb49abf99c18bc1c12363d898f3ced7 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527556745 +0800	commit: add lib/index.js

```

git log

```sh
commit 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 (HEAD -> master)
Author: liuyanjie <x@gmail.com>
Date:   Tue May 29 09:19:05 2018 +0800

    add lib/index.js

commit 28baf4f77fb49abf99c18bc1c12363d898f3ced7
Author: liuyanjie <x@gmail.com>
Date:   Tue May 29 00:20:52 2018 +0800

    add CHANGELOG.md and CONTRIBUTING.md files

commit 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 (tag: v0.0.1)
Author: liuyanjie <x@gmail.com>
Date:   Mon May 28 23:38:14 2018 +0800

    First Commmit

```

### 开始创建分支

创建并切换到分支 `feature-a`

```sh
$ git branch -v

* master 2b9fc85 add lib/index.js

$ git checkout -b feature-a
Switched to a new branch 'feature-a'

# liuyanjie @ bmw in ~/git-obj-model on git:feature-a o [9:25:01]
$ git branch -v

* feature-a 2b9fc85 add lib/index.js
  master    2b9fc85 add lib/index.js

$ git status
On branch feature-a
nothing to commit, working tree clean

```

```sh
$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           ├── feature-a
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5
│   ├── 28
│   │   └── baf4f77fb49abf99c18bc1c12363d898f3ced7
│   ├── 2b
│   │   └── 9fc8524bac21d5d5c2f988b5793315ce93abc6
│   ├── 2c
│   │   └── affd90cd736e58f516e3988e3af84f5fa42b4f
│   ├── 2f
│   │   └── b9045bb558889ea2bd8cc5d8fe45e7247706da
│   ├── 39
│   │   └── a204af28de9b4f0411735e597e0da7416ca35a
│   ├── 96
│   │   └── 09811e44367d44f2915435f4454716e1e535fd
│   ├── a0
│   │   └── cf709bc0991b5340080f944d02894dc1596d46
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe
│   ├── c6
│   │   └── b9e95b39b8cd8ead8bbf4b118104741017de1b
│   ├── cc
│   │   └── f88a6f1649213499841c33f9bb36d1d8756fb7
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   ├── feature-a
    │   └── master
    └── tags
        └── v0.0.1

23 directories, 35 files

```

看一下`.git`文件内容，可以看到`.git/refs/heads`目录下多了个`feature-a`

查看一下 `feature-a` 的相关内容，指向的提交对象和 `master` 一样，并指明 `Created from HEAD`。

从内容中可以看出，两个分支指向同一个提交对象 `1ad0e5`，但是两个分支的日志不同。

日志记录了分支的历史，而提交对象记录了分支的数据内容和所有分支的历史。

```sh
$ cat .git/refs/heads/feature-a
2b9fc8524bac21d5d5c2f988b5793315ce93abc6

$ cat .git/refs/heads/master
2b9fc8524bac21d5d5c2f988b5793315ce93abc6
```

```sh
$ cat .git/logs/refs/heads/feature-a
0000000000000000000000000000000000000000 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527557101 +0800	branch: Created from HEAD

$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit
033fa1ab71f0d54f348f07a3a0ffcefd52804df5 28baf4f77fb49abf99c18bc1c12363d898f3ced7 liuyanjie <x@gmail.com> 1527524452 +0800	commit: add CHANGELOG.md and CONTRIBUTING.md files
28baf4f77fb49abf99c18bc1c12363d898f3ced7 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527556745 +0800	commit: add lib/index.js
```

初始化`package.json`

```sh
$ npm init
Is this ok? (yes)
```

安装 `bluebird`

```sh
$ npm install bluebird --save
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN git-obj-model@1.0.0 No description
npm WARN git-obj-model@1.0.0 No repository field.

+ bluebird@3.5.1
added 1 package in 1.999s

```

添加文件到版本库，并查看变化。

```sh
$ git add package.json

$ cat .git/refs/heads/feature-a
2b9fc8524bac21d5d5c2f988b5793315ce93abc6

$ cat .git/refs/heads/master
2b9fc8524bac21d5d5c2f988b5793315ce93abc6

```

提交文件到版本库，并查看变化，提交指针已经指向新的提交对象，分支的版本超前于master，因为parent指向`1ad0e5`

```sh
$ git commit package.json -m 'add package.json'
[feature-a 74ce192] add package.json
 1 file changed, 17 insertions(+)
 create mode 100644 package.json

$ cat .git/refs/heads/master
2b9fc8524bac21d5d5c2f988b5793315ce93abc6

$ cat .git/refs/heads/feature-a
74ce19245be785772b33e1193df0d2c6a940ed10

$ git cat-file -p 74ce
tree 9d23f8ad2ecc9f8e49d78174bcc6e4668ca7f031
parent 2b9fc8524bac21d5d5c2f988b5793315ce93abc6
author liuyanjie <x@gmail.com> 1527557656 +0800
committer liuyanjie <x@gmail.com> 1527557656 +0800

add package.json

$ git cat-file -p 9d23
100644 blob a0cf709bc0991b5340080f944d02894dc1596d46	CHANGELOG.md
100644 blob c6b9e95b39b8cd8ead8bbf4b118104741017de1b	CONTRIBUTING.md
100644 blob f3954314c1026028e77ea3a765aadefa67b45195	README.md
040000 tree 2fb9045bb558889ea2bd8cc5d8fe45e7247706da	lib
100644 blob 5ee275ce022007c38d187c58aef13f8d9e04b553	package.json

```

合并分支 `feature-a` 到 `master` 分支，可以看到 `master` 分支跟上了 `feature-a`，只改指针既可，非常快。

```sh
$ git checkout master
Switched to branch 'master'

$ git merge feature-a --no-ff
Merge made by the 'recursive' strategy.
 package.json | 17 +++++++++++++++++
 1 file changed, 17 insertions(+)
 create mode 100644 package.json

$ cat .git/refs/heads/master
b98c36254dba6e169429c2be57b1bbeccb6e1f28

$ cat .git/refs/heads/feature-a
74ce19245be785772b33e1193df0d2c6a940ed10

```

```sh
$ tig

2018-05-29 09:37 liuyanjie M─┐ [master] Merge branch 'feature-a'
2018-05-29 09:34 liuyanjie │ o [feature-a] add package.json
2018-05-29 09:19 liuyanjie o─┘ add lib/index.js
2018-05-29 00:20 liuyanjie o add CHANGELOG.md and CONTRIBUTING.md files
2018-05-28 23:38 liuyanjie I <v0.0.1> First Commmit

```

修改 `master` 分支，编辑文件并提交

```sh
$ git checkout master
Already on 'master'

$ echo -e "\n\n# About" >> README.md

$ git commit README.md -m "add About Me"
[master cc52745] add About Me
 1 file changed, 4 insertions(+)

$ cat README.md
# Readme


# About
```

修改 `feature-a` 分支

```sh
$ git checkout feature-a
Switched to branch 'feature-a'

$ cat README.md
# Readme

$ echo -e "\n\n# About" >> README.md

$ cat README.md
# Readme


# About

$ git add README.md

$ git commit README.md -m "add About Me"
[feature-a fd885cf] add About Me
 1 file changed, 3 insertions(+)

```

查看分支头和历史记录，可以看到分支头指向不同的提交对象，而提交对象，又来源于同一个提交对象`0cb214741c044af3fb4677fe72e3ae175f3e0358`，两个分支之间存在交叉。

```sh
$ cat .git/refs/heads/feature-a
74ce19245be785772b33e1193df0d2c6a940ed10

$ cat .git/refs/heads/master
cc52745b13b00c672c7ac9b1dc42336953293b7a

$ cat .git/logs/refs/heads/feature-a
0000000000000000000000000000000000000000 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527557101 +0800	branch: Created from HEAD
2b9fc8524bac21d5d5c2f988b5793315ce93abc6 74ce19245be785772b33e1193df0d2c6a940ed10 liuyanjie <x@gmail.com> 1527557656 +0800	commit: add package.json
74ce19245be785772b33e1193df0d2c6a940ed10 fd885cfcd548a24fabb2f5b49ea60beadc8c5912 liuyanjie <x@gmail.com> 1527558668 +0800	commit: add About Me

$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit
033fa1ab71f0d54f348f07a3a0ffcefd52804df5 28baf4f77fb49abf99c18bc1c12363d898f3ced7 liuyanjie <x@gmail.com> 1527524452 +0800	commit: add CHANGELOG.md and CONTRIBUTING.md files
28baf4f77fb49abf99c18bc1c12363d898f3ced7 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527556745 +0800	commit: add lib/index.js
2b9fc8524bac21d5d5c2f988b5793315ce93abc6 b98c36254dba6e169429c2be57b1bbeccb6e1f28 liuyanjie <x@gmail.com> 1527557853 +0800	merge feature-a: Merge made by the 'recursive' strategy.
b98c36254dba6e169429c2be57b1bbeccb6e1f28 cc52745b13b00c672c7ac9b1dc42336953293b7a liuyanjie <x@gmail.com> 1527558514 +0800	commit: add About Me

```

切换到 `master` 分支，再次合并 `feature-a` 到 `master`

```sh
$ git checkout master
Switched to branch 'master'

$ git merge feature-a
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.

$ cat README.md
# Readme


<<<<<<< HEAD
## About

=======
## About
>>>>>>> feature-a

```

解决冲突之后的文件

```sh
$ cat README.md
# Readme


## About


```

```sh
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)

	both modified:   README.md

```

```sh
$ git commit README.md -m 'fix conflict'
fatal: cannot do a partial commit during a merge.

$ git commit -a
[master 5a68c74] Merge branch 'feature-a'
```

```sh
$ cat .git/refs/heads/feature-a
fd885cfcd548a24fabb2f5b49ea60beadc8c5912

$ cat .git/refs/heads/master
5a68c7402c30ae44fd82dadba2d8f148efb75541
```

```sh
$ cat .git/logs/refs/heads/master
0000000000000000000000000000000000000000 033fa1ab71f0d54f348f07a3a0ffcefd52804df5 liuyanjie <x@gmail.com> 1527521894 +0800	commit (initial): First Commmit
033fa1ab71f0d54f348f07a3a0ffcefd52804df5 28baf4f77fb49abf99c18bc1c12363d898f3ced7 liuyanjie <x@gmail.com> 1527524452 +0800	commit: add CHANGELOG.md and CONTRIBUTING.md files
28baf4f77fb49abf99c18bc1c12363d898f3ced7 2b9fc8524bac21d5d5c2f988b5793315ce93abc6 liuyanjie <x@gmail.com> 1527556745 +0800	commit: add lib/index.js
2b9fc8524bac21d5d5c2f988b5793315ce93abc6 b98c36254dba6e169429c2be57b1bbeccb6e1f28 liuyanjie <x@gmail.com> 1527557853 +0800	merge feature-a: Merge made by the 'recursive' strategy.
b98c36254dba6e169429c2be57b1bbeccb6e1f28 cc52745b13b00c672c7ac9b1dc42336953293b7a liuyanjie <x@gmail.com> 1527558514 +0800	commit: add About Me
cc52745b13b00c672c7ac9b1dc42336953293b7a 5a68c7402c30ae44fd82dadba2d8f148efb75541 liuyanjie <x@gmail.com> 1527559277 +0800	commit (merge): Merge branch 'feature-a'

```

```sh
$ git merge --stat feature-a
Already up to date.

$ git branch -d feature-a
Deleted branch feature-a (was fd885cf).

$ git branch

$ git cat-file -p fd88
tree 82e47e13f87031cab7c47b455744bc4ed3fdba3c
parent 74ce19245be785772b33e1193df0d2c6a940ed10
author liuyanjie <x@gmail.com> 1527558668 +0800
committer liuyanjie <x@gmail.com> 1527558668 +0800

add About Me

$ git cat-file -p 5a68
tree 42564bab966d59e3501a1903c1cb2066adc2bd2d
parent cc52745b13b00c672c7ac9b1dc42336953293b7a
parent fd885cfcd548a24fabb2f5b49ea60beadc8c5912
author liuyanjie <x@gmail.com> 1527559277 +0800
committer liuyanjie <x@gmail.com> 1527559277 +0800

Merge branch 'feature-a'
```

```sh
$ git log --graph --all --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative
*   5a68c74 - (HEAD -> master) Merge branch 'feature-a' (7 minutes ago) <liuyanjie>
|\
| * fd885cf - add About Me (17 minutes ago) <liuyanjie>
* | cc52745 - add About Me (20 minutes ago) <liuyanjie>
* |   b98c362 - Merge branch 'feature-a' (31 minutes ago) <liuyanjie>
|\ \
| |/
| * 74ce192 - add package.json (34 minutes ago) <liuyanjie>
|/
* 2b9fc85 - add lib/index.js (49 minutes ago) <liuyanjie>
* 28baf4f - add CHANGELOG.md and CONTRIBUTING.md files (10 hours ago) <liuyanjie>
* 033fa1a - (tag: v0.0.1) First Commmit (11 hours ago) <liuyanjie>
```

配置命令别名

```sh
$ git config --global alias.lg "log --graph --all --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative"
git lg
```

### 打标签

```sh
$ git tag -a v0.0.2 -m "v0.0.2"
v0.0.1
v0.0.2

$ cat .git/refs/tags/v0.0.2
174c530154ab3b4d4d322bcbd66cdde18882eb00

$ git cat-file -p 174c
object 5a68c7402c30ae44fd82dadba2d8f148efb75541
type commit
tag v0.0.2
tagger liuyanjie <x@gmail.com> 1527559851 +0800

v0.0.2

$ git cat-file -p 5a68
tree 42564bab966d59e3501a1903c1cb2066adc2bd2d
parent cc52745b13b00c672c7ac9b1dc42336953293b7a
parent fd885cfcd548a24fabb2f5b49ea60beadc8c5912
author liuyanjie <x@gmail.com> 1527559277 +0800
committer liuyanjie <x@gmail.com> 1527559277 +0800

Merge branch 'feature-a'
```

```sh
$ tree .git
.git
├── COMMIT_EDITMSG
├── HEAD
├── ORIG_HEAD
├── co.gitup.mac
│   ├── cache.db
│   ├── info.plist
│   └── snapshots.data
├── config
├── description
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   ├── prepare-commit-msg.sample
│   └── update.sample
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 03
│   │   └── 3fa1ab71f0d54f348f07a3a0ffcefd52804df5
│   ├── 17
│   │   └── 4c530154ab3b4d4d322bcbd66cdde18882eb00
│   ├── 28
│   │   └── baf4f77fb49abf99c18bc1c12363d898f3ced7
│   ├── 2b
│   │   └── 9fc8524bac21d5d5c2f988b5793315ce93abc6
│   ├── 2c
│   │   └── affd90cd736e58f516e3988e3af84f5fa42b4f
│   ├── 2f
│   │   └── b9045bb558889ea2bd8cc5d8fe45e7247706da
│   ├── 30
│   │   └── be8054b61a4a124e4cd8a1201de8474cb45e12
│   ├── 39
│   │   └── a204af28de9b4f0411735e597e0da7416ca35a
│   ├── 42
│   │   └── 564bab966d59e3501a1903c1cb2066adc2bd2d
│   ├── 44
│   │   └── 2b821b9b4acb5b0e632042542b3b29e2cf721e
│   ├── 48
│   │   └── 1cab7467cdb697bf7affdba0f2a5673a03366d
│   ├── 5a
│   │   └── 68c7402c30ae44fd82dadba2d8f148efb75541
│   ├── 5e
│   │   └── e275ce022007c38d187c58aef13f8d9e04b553
│   ├── 74
│   │   └── ce19245be785772b33e1193df0d2c6a940ed10
│   ├── 82
│   │   └── e47e13f87031cab7c47b455744bc4ed3fdba3c
│   ├── 96
│   │   └── 09811e44367d44f2915435f4454716e1e535fd
│   ├── 9d
│   │   └── 23f8ad2ecc9f8e49d78174bcc6e4668ca7f031
│   ├── a0
│   │   └── cf709bc0991b5340080f944d02894dc1596d46
│   ├── b9
│   │   └── 8c36254dba6e169429c2be57b1bbeccb6e1f28
│   ├── c1
│   │   └── 067ba0a7ba51f937518c9bc051ea744ca748fe
│   ├── c6
│   │   └── b9e95b39b8cd8ead8bbf4b118104741017de1b
│   ├── cc
│   │   ├── 52745b13b00c672c7ac9b1dc42336953293b7a
│   │   └── f88a6f1649213499841c33f9bb36d1d8756fb7
│   ├── f3
│   │   └── 954314c1026028e77ea3a765aadefa67b45195
│   ├── fd
│   │   └── 885cfcd548a24fabb2f5b49ea60beadc8c5912
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags
        ├── v0.0.1
        └── v0.0.2

36 directories, 51 files

```

https://stackoverflow.com/questions/964876/head-and-orig-head-in-git

.git/HEAD 指明了当前活跃分支是哪个分支

```sh
$ cat .git/HEAD
ref: refs/heads/master

```

[svg版](https://github.com/liuyanjie/knowledge/tree/master/vcs/git/images/git-obj-model.svg)

![git-obj-model](https://raw.githubusercontent.com/liuyanjie/knowledge/master/vcs/git/images/git-obj-model.png)


---

<a href="https://github.com/liuyanjie/knowledge/tree/master/vcs/git/git-object-model.md" >查看源文件</a>&nbsp;&nbsp;<a href="https://github.com/liuyanjie/knowledge/edit/master/vcs/git/git-object-model.md">编辑源文件</a>
