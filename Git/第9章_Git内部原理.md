# 第9章_Git内部原理

从根本上来讲 Git 是一个内容寻址（content-addressable）文件系统，并在此之上提供了一个版本控制系统的用户界面。

## 1.底层命令与上层命令

本书主要涵盖了`checkout`、`branch`、`remote`等约 30 个 Git 的子命令。 然而，由于 Git 最初是一套面向版本控制系统的工具集，而不是一个完整的、用户友好的版本控制系统， 所以它还包含了一部分用于完成底层工作的子命令。 这些命令被设计成能以 UNIX 命令行的风格连接在一起，抑或藉由脚本调用，来完成工作。 这部分命令一般被称作“底层（plumbing）”命令，而那些更友好的命令则被称作“上层（porcelain）”命令。

当在一个新目录或已有目录执行`git init`时，Git 会创建一个 .git 目录。 这个目录包含了几乎所有 Git 存储和操作的东西。 如若想备份或复制一个版本库，只需把这个目录拷贝至另一处即可。 本章探讨的所有内容，均位于这个目录内。 新初始化的 .git 目录的典型结构如下：

```bash
$ ls -F1
config
description
HEAD
hooks/
info/
objects/
refs/
```

随着 Git 版本的不同，该目录下可能还会包含其他内容。 不过对于一个全新的`git init`版本库，这将是你看到的默认结构。**description**文件仅供 GitWeb 程序使用，我们无需关心。**config**文件包含项目特有的配置选项。**info**目录包含一个全局性排除（global exclude）文件，用以放置那些不希望被记录在 .gitignore文件中的忽略模式（ignored patterns）。**hooks**目录包含客户端或服务端的钩子脚本（hook scripts）。

剩下的四个条目很重要：**HEAD**文件、（尚待创建的）**index**文件，和**objects**目录、**refs**目录。 它们都是 Git 的核心组成部分。

**objects**目录存储所有数据内容；**refs**目录存储指向数据（分支、远程仓库和标签等）的提交对象的指针；**HEAD**文件指向目前被检出的分支；**index**文件保存暂存区信息。 我们将详细地逐一检视这四部分，来理解 Git 是如何运转的。

## 2.Git对象

Git 的核心部分是一个简单的**键值对数据库**。 你可以向 Git 仓库中插入任意类型的内容，它会返回一个唯一的键，通过该键可以在任意时刻再次取回该内容。

可以通过底层命令`git hash-object`来演示上述效果——该命令可将任意数据保存于 .git/objects 目录（即对象数据库），并返回指向该数据对象的唯一的键。

首先，我们需要初始化一个新的 Git 版本库，并确认 objects 目录为空：

```bash
$ git init test
Initialized empty Git repository in /tmp/test/.git/
$ cd test
$ find .git/objects
.git/objects
.git/objects/info
.git/objects/pack
$ find .git/objects -type f
```

可以看到 Git 对 objects 目录进行了初始化，并创建了 pack 和 info 子目录，但均为空。 接着，我们用`git hash-object`创建一个新的数据对象并将它手动存入你的新 Git 数据库中：

```bash
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

在这种最简单的形式中，`git hash-object`会接受你传给它的东西，而它只会返回可以存储在 Git 仓库中的唯一键。`-w`选项会指示该命令不要只返回键，还要将该对象写入数据库中。最后，`--stdin`选项则指示该命令从标准输入读取内容；若不指定此选项，则须在命令尾部给出待存储文件的路径。

此命令输出一个长度为 40 个字符的校验和。 这是一个 SHA-1 哈希值——一个将待存储的数据外加一个头部信息（header）一起做 SHA-1 校验运算而得的校验和。后文会简要讨论该头部信息。 现在我们可以查看 Git 是如何存储数据的：

```bash
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

如果你再次查看 objects 目录，那么可以在其中找到一个与新内容对应的文件。 这就是开始时 Git 存储内容的方式——一个文件对应一条内容， 以该内容加上特定头部信息一起的 SHA-1 校验和为文件命名。 校验和的前两个字符用于命名子目录，余下的 38 个字符则用作文件名。

一旦你将内容存储在了对象数据库中，那么可以通过`cat-file`命令从 Git 那里取回数据。 这个命令简直就是一把剖析 Git 对象的瑞士军刀。 为`cat-file`指定`-p`选项可指示该命令自动判断内容的类型，并为我们显示大致的内容：

```bash
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
test content
```

至此，你已经掌握了如何向 Git 中存入内容，以及如何将它们取出。 我们同样可以将这些操作应用于文件中的内容。 例如，可以对一个文件进行简单的版本控制。 首先，创建一个新文件并将其内容存入数据库：

```bash
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30
```

接着，向文件里写入新内容，并再次将其存入数据库：

```bash
$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
```

对象数据库记录下了该文件的两个不同版本，当然之前我们存入的第一条内容也还在：

```bash
$ find .git/objects -type f
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```

现在可以在删掉 test.txt 的本地副本，然后用 Git 从对象数据库中取回它的第一个版本：

```bash
$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30 > test.txt
$ cat test.txt
version 1
```

或者第二个版本：

```bash
$ git cat-file -p 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a > test.txt
$ cat test.txt
version 2
```

然而，记住文件的每一个版本所对应的 SHA-1 值并不现实；另一个问题是，在这个（简单的版本控制）系统中，文件名并没有被保存——我们仅保存了文件的内容。 上述类型的对象我们称之为**数据对象**（blob object）。利用`git cat-file -t`命令，可以让 Git 告诉我们其内部存储的任何对象类型，只要给定该对象的 SHA-1 值：

```bash
$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
blob
```

### 2.1 树对象

接下来要探讨的 Git 对象类型是树对象（tree object），它能解决文件名保存的问题，也允许我们将多个文件组织到一起。 Git 以一种类似于 UNIX 文件系统的方式存储内容，但作了些许简化。 所有内容均以树对象和数据对象的形式存储，其中树对象对应了 UNIX 中的目录项，数据对象则大致上对应了 inodes 或文件内容。 一个树对象包含了一条或多条树对象记录（tree entry），每条记录含有一个指向数据对象或者子树对象的 SHA-1 指针，以及相应的模式、类型、文件名信息。 例如，某项目当前对应的最新树对象可能是这样的：

```bash
$ git cat-file -p master^{tree}
100644 blob a906cb2a4a904a152e80877d4088654daad0c859      README
100644 blob 8f94139338f9404f26296befa88755fc2598c289      Rakefile
040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0      lib
```

`master^{tree}`语法表示 master 分支上最新的提交所指向的树对象。 请注意，lib 子目录（所对应的那条树对象记录）并不是一个数据对象，而是一个指针，其指向的是另一个树对象：

```bash
$ git cat-file -p 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0
100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b      simplegit.rb
```

> 你可能会在某些 shell 中使用`master^{tree}`语法时遇到错误。
>
> 在 Windows 的 CMD 中，字符`^`被用于转义，因此你必须双写它以避免出现问题：`git cat-file -p master^^{tree}`。 在 PowerShell 中使用字符 {} 时则必须用引号引起来，以此来避免参数解析错误：`git cat-file -p 'master^{tree}'`。
>
> 在 ZSH 中，字符`^`被用在通配模式（globbing）中，因此你必须将整个表达式用引号引起来：`git cat-file -p "master^{tree}"`。

从概念上讲，Git 内部存储的数据有点像这样：

<img src="img/第9章_Git内部原理/image-20220708203017264.png" alt="image-20220708203017264" style="zoom:33%;" />

你可以轻松创建自己的树对象。 通常，Git 根据某一时刻暂存区（即 index 区域，下同）所表示的状态创建并记录一个对应的树对象， 如此重复便可依次记录（某个时间段内）一系列的树对象。 因此，为创建一个树对象，首先需要通过暂存一些文件来创建一个暂存区。 可以通过底层命令`git update-index`为一个单独文件——我们的 test.txt 文件的首个版本——创建一个暂存区。 利用该命令，可以把 test.txt 文件的首个版本人为地加入一个新的暂存区。 必须为上述命令指定`--add`选项，因为此前该文件并不在暂存区中（我们甚至都还没来得及创建一个暂存区呢）； 同样必需的还有`--cacheinfo`选项，因为将要添加的文件位于 Git 数据库中，而不是位于当前目录下。 同时，需要指定文件模式、SHA-1 与文件名：

```bash
$ git update-index --add --cacheinfo 100644 83baae61804e65cc73a7201a7252750c76066a30 test.txt
```

本例中，我们指定的文件模式为`100644`，表明这是一个普通文件。 其他选择包括：`100755`，表示一个可执行文件；`120000`，表示一个符号链接。 这里的文件模式参考了常见的 UNIX 文件模式，但远没那么灵活——上述三种模式即是 Git 文件（即数据对象）的所有合法模式（当然，还有其他一些模式，但用于目录项和子模块）。

现在，可以通过`git write-tree`命令将暂存区内容写入一个树对象。 此处无需指定`-w`选项——如果某个树对象此前并不存在的话，当调用此命令时， 它会根据当前暂存区状态自动创建一个新的树对象：

```bash
$ git write-tree
d8329fc1cc938780ffdd9f94e0d364e0ea74f579
$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
100644 blob 83baae61804e65cc73a7201a7252750c76066a30      test.txt
```

不妨用之前见过的`git cat-file`命令验证一下它确实是一个树对象：

```bash
$ git cat-file -t d8329fc1cc938780ffdd9f94e0d364e0ea74f579
tree
```

接着我们来创建一个新的树对象，它包括 test.txt 文件的第二个版本，以及一个新的文件：

```bash
$ echo 'new file' > new.txt
$ git update-index --add --cacheinfo 100644 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a test.txt
$ git update-index --add new.txt
```

暂存区现在包含了 test.txt 文件的新版本，和一个新文件：new.txt。 记录下这个目录树（将当前暂存区的状态记录为一个树对象），然后观察它的结构：

```bash
$ git write-tree
0155eb4229851634a0f03eb265b69f5a2d56f341
$ git cat-file -p 0155eb4229851634a0f03eb265b69f5a2d56f341
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt
```

我们注意到，新的树对象包含两条文件记录，同时 test.txt 的 SHA-1 值（1f7a7a）是先前值的 “version 2”。 只是为了好玩：你可以将第一个树对象加入第二个树对象，使其成为新的树对象的一个子目录。 通过调用`git read-tree`命令，可以把树对象读入暂存区。 本例中，可以通过对该命令指定`--prefix`选项，将一个已有的树对象作为子树读入暂存区：

```bash
$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579
$ git write-tree
3c4e9cd789d88d8d89c1073707c3585e41b0e614
$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt
```

如果基于这个新的树对象创建一个工作目录，你会发现工作目录的根目录包含两个文件以及一个名为 bak 的子目录，该子目录包含 test.txt 文件的第一个版本。 可以认为 Git 内部存储着的用于表示上述结构的数据是这样的：

<img src="img/第9章_Git内部原理/image-20220708203657008.png" alt="image-20220708203657008" style="zoom:33%;" />

### 2.2 提交对象

如果你做完了以上所有操作，那么现在就有了三个树对象，分别代表我们想要跟踪的不同项目快照。 然而问题依旧：若想重用这些快照，你必须记住所有三个 SHA-1 哈希值。 并且，你也完全不知道是谁保存了这些快照，在什么时刻保存的，以及为什么保存这些快照。 而以上这些，正是提交对象（commit object）能为你保存的基本信息。

可以通过调用`commit-tree`命令创建一个提交对象，为此需要指定一个树对象的 SHA-1 值，以及该提交的父提交对象（如果有的话）。 我们从之前创建的第一个树对象开始：

```bash
$ echo 'first commit' | git commit-tree d8329f
fdf4fc3344e67ab068f836878b6c4951e3b15f3d
```

由于创建时间和作者数据不同，你现在会得到一个不同的散列值。 请将本章后续内容中的提交和标签的散列值替换为你自己的校验和。 现在可以通过`git cat-file`命令查看这个新提交对象：

```bash
$ git cat-file -p fdf4fc3
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
author Scott Chacon <schacon@gmail.com> 1243040974 -0700
committer Scott Chacon <schacon@gmail.com> 1243040974 -0700

first commit
```

提交对象的格式很简单：它先指定一个顶层树对象，代表当前项目快照； 然后是可能存在的父提交（前面描述的提交对象并不存在任何父提交）； 之后是作者/提交者信息（依据你的 user.name 和 user.email 配置来设定，外加一个时间戳）； 留空一行，最后是提交注释。

接着，我们将创建另两个提交对象，它们分别引用各自的上一个提交（作为其父提交对象）：

```bash
$ echo 'second commit' | git commit-tree 0155eb -p fdf4fc3
cac0cab538b970a37ea1e769cbbde608743bc96d
$ echo 'third commit'  | git commit-tree 3c4e9c -p cac0cab
1a410efbd13591db07496601ebc7a059dd55cfe9
```

这三个提交对象分别指向之前创建的三个树对象快照中的一个。 现在，如果对最后一个提交的 SHA-1 值运行`git log`命令，会出乎意料的发现，你已有一个货真价实的、可由`git log`查看的 Git 提交历史了：

```bash
$ git log --stat 1a410e
commit 1a410efbd13591db07496601ebc7a059dd55cfe9
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:15:24 2009 -0700

    third commit

 bak/test.txt | 1 +
 1 file changed, 1 insertion(+)

commit cac0cab538b970a37ea1e769cbbde608743bc96d
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:14:29 2009 -0700

    second commit

 new.txt  | 1 +
 test.txt | 2 +-
 2 files changed, 2 insertions(+), 1 deletion(-)

commit fdf4fc3344e67ab068f836878b6c4951e3b15f3d
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:09:34 2009 -0700

    first commit

 test.txt | 1 +
 1 file changed, 1 insertion(+)
```

太神奇了： 就在刚才，你没有借助任何上层命令，仅凭几个底层操作便完成了一个 Git 提交历史的创建。 这就是每次我们运行`git add`和`git commit`命令时，Git 所做的工作实质就是将被改写的文件保存为数据对象，更新暂存区，记录树对象，最后创建一个指明了顶层树对象和父提交的提交对象。 这三种主要的 Git 对象——数据对象、树对象、提交对象——最初均以单独文件的形式保存在 .git/objects 目录下。 下面列出了目前示例目录内的所有对象，辅以各自所保存内容的注释：

```bash
$ find .git/objects -type f
.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1
```

如果跟踪所有的内部指针，将得到一个类似下面的对象关系图：

<img src="img/第9章_Git内部原理/image-20220708204056218.png" alt="image-20220708204056218" style="zoom:33%;" />

### 2.3 对象存储

前文曾提及，你向 Git 仓库提交的所有对象都会有个头部信息一并被保存。 让我们略花些时间来看看 Git 是如何存储其对象的。 通过在 Ruby 脚本语言中交互式地演示，你将看到一个数据对象——本例中是字符串 “what is up, doc?” 是如何被存储的。

可以通过 irb 命令启动 Ruby 的交互模式：

```ruby
$ irb
>> content = "what is up, doc?"
=> "what is up, doc?"
```

Git 首先会以识别出的对象的类型作为开头来构造一个头部信息，本例中是一个“blob”字符串。 接着 Git 会在头部的第一部分添加一个空格，随后是数据内容的字节数，最后是一个空字节（null byte）：

```ruby
>> header = "blob #{content.length}\0"
=> "blob 16\u0000"
```

Git 会将上述头部信息和原始数据拼接起来，并计算出这条新内容的 SHA-1 校验和。 在 Ruby 中可以这样计算 SHA-1 值——先通过 require 命令导入 SHA-1 digest 库， 然后对目标字符串调用`Digest::SHA1.hexdigest()`：

```ruby
>> store = header + content
=> "blob 16\u0000what is up, doc?"
>> require 'digest/sha1'
=> true
>> sha1 = Digest::SHA1.hexdigest(store)
=> "bd9dbf5aae1a3862dd1526723246b20206e5fc37"
```

我们来比较一下`git hash-object`的输出。 这里使用了`echo -n`以避免在输出中添加换行。

```bash
$ echo -n "what is up, doc?" | git hash-object --stdin
bd9dbf5aae1a3862dd1526723246b20206e5fc37
```

Git 会通过 zlib 压缩这条新内容。在 Ruby 中可以借助 zlib 库做到这一点。 先导入相应的库，然后对目标内容调用`Zlib::Deflate.deflate()`：

```ruby
>> require 'zlib'
=> true
>> zlib_content = Zlib::Deflate.deflate(store)
=> "x\x9CK\xCA\xC9OR04c(\xCFH,Q\xC8,V(-\xD0QH\xC9O\xB6\a\x00_\x1C\a\x9D"
```

最后，需要将这条经由 zlib 压缩的内容写入磁盘上的某个对象。 要先确定待写入对象的路径（SHA-1 值的前两个字符作为子目录名称，后 38 个字符则作为子目录内文件的名称）。 如果该子目录不存在，可以通过 Ruby 中的`FileUtils.mkdir_p()`函数来创建它。 接着，通过 File.open() 打开这个文件。最后，对上一步中得到的文件句柄调用 write() 函数，以向目标文件写入之前那条 zlib 压缩过的内容：

```ruby
>> path = '.git/objects/' + sha1[0,2] + '/' + sha1[2,38]
=> ".git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37"
>> require 'fileutils'
=> true
>> FileUtils.mkdir_p(File.dirname(path))
=> ".git/objects/bd"
>> File.open(path, 'w') { |f| f.write zlib_content }
=> 32
```

我们用`git cat-file`查看一下该对象的内容：

```bash
---
$ git cat-file -p bd9dbf5aae1a3862dd1526723246b20206e5fc37
what is up, doc?
---
```

就是这样——你已创建了一个有效的 Git 数据对象。

所有的 Git 对象均以这种方式存储，区别仅在于类型标识——另两种对象类型的头部信息以字符串“commit”或“tree”开头，而不是“blob”。 另外，虽然数据对象的内容几乎可以是任何东西，但提交对象和树对象的内容却有各自固定的格式。

## 3.Git引用

如果你对仓库中从一个提交（比如 1a410e）开始往前的历史感兴趣，那么可以运行`git log 1a410e`这样的命令来显示历史，不过你需要记得 1a410e 是你查看历史的起点提交。 如果我们有一个文件来保存 SHA-1 值，而该文件有一个简单的名字， 然后用这个名字指针来替代原始的 SHA-1 值的话会更加简单。

在 Git 中，这种简单的名字被称为**引用**（references，或简写为 refs）。 你可以在 .git/refs 目录下找到这类含有 SHA-1 值的文件。 在目前的项目中，这个目录没有包含任何文件，但它包含了一个简单的目录结构：

```bash
$ find .git/refs
.git/refs
.git/refs/heads
.git/refs/tags
$ find .git/refs -type f
```

若要创建一个新引用来帮助记忆最新提交所在的位置，从技术上讲我们只需简单地做如下操作：

```bash
$ echo 1a410efbd13591db07496601ebc7a059dd55cfe9 > .git/refs/heads/master
```

现在，你就可以在 Git 命令中使用这个刚创建的新引用来代替 SHA-1 值了：

```bash
$ git log --pretty=oneline master
1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
cac0cab538b970a37ea1e769cbbde608743bc96d second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit
```

我们不提倡直接编辑引用文件。 如果想更新某个引用，Git 提供了一个更加安全的命令`update-ref`来完成此事：

```bash
$ git update-ref refs/heads/master
1a410efbd13591db07496601ebc7a059dd55cfe9
```

这基本就是 Git 分支的本质：一个指向某一系列提交之首的指针或引用。 若想在第二个提交上创建一个分支，可以这么做：

```bash
$ git update-ref refs/heads/test cac0ca
```

这个分支将只包含从第二个提交开始往前追溯的记录：

```bash
$ git log --pretty=oneline test
cac0cab538b970a37ea1e769cbbde608743bc96d second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit
```

至此，我们的 Git 数据库从概念上看起来像这样：

<img src="img/第9章_Git内部原理/image-20220708205744616.png" alt="image-20220708205744616" style="zoom:33%;" />

当运行类似于`git branch <branch>`这样的命令时，Git 实际上会运行`update-ref`命令， 取得当前所在分支最新提交对应的 SHA-1 值，并将其加入你想要创建的任何新引用中。

### 3.1 HEAD引用

现在的问题是，当你执行 git branch <branch> 时，Git 如何知道最新提交的 SHA-1 值呢？ 答案是 HEAD 文件。

HEAD 文件通常是一个符号引用（symbolic reference），指向目前所在的分支。所谓符号引用，表示它是一个指向其他引用的指针。

然而在某些罕见的情况下，HEAD 文件可能会包含一个 git 对象的 SHA-1 值。 当你在检出一个标签、提交或远程分支，让你的仓库变成[分离 HEAD](https://git-scm.com/docs/git-checkout#_detached_head)状态时，就会出现这种情况。

如果查看 HEAD 文件的内容，通常我们看到类似这样的内容：

```bash
$ cat .git/HEAD
ref: refs/heads/master
```

如果执行`git checkout test`，Git 会像这样更新 HEAD 文件：

```bash
$ cat .git/HEAD
ref: refs/heads/test
```

当我们执行`git commit`时，该命令会创建一个提交对象，并用 HEAD 文件中那个引用所指向的 SHA-1 值设置其父提交字段。

你也可以手动编辑该文件，然而同样存在一个更安全的命令来完成此事：`git symbolic-ref`。 可以借助此命令来查看 HEAD 引用对应的值：

```bash
$ git symbolic-ref HEAD
refs/heads/master
```

同样可以设置 HEAD 引用的值：

```bash
$ git symbolic-ref HEAD refs/heads/test
$ cat .git/HEAD
ref: refs/heads/test
```

不能把符号引用设置为一个不符合引用规范的值：

```bash
$ git symbolic-ref HEAD test
fatal: Refusing to point HEAD outside of refs/
```

### 3.2 标签引用

前面我们刚讨论过 Git 的三种主要的对象类型（数据对象、树对象 和 提交对象 ），然而实际上还有第四种。 标签对象（tag object） 非常类似于一个提交对象——它包含一个标签创建者信息、一个日期、一段注释信息，以及一个指针。 主要的区别在于，标签对象通常指向一个提交对象，而不是一个树对象。 它像是一个永不移动的分支引用——永远指向同一个提交对象，只不过给这个提交对象加上一个更友好的名字罢了。

存在两种类型的标签：附注标签和轻量标签。 可以像这样创建一个轻量标签：

```bash
$ git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d
```

这就是轻量标签的全部内容，一个固定的引用。 然而，一个附注标签则更复杂一些。 若要创建一个附注标签，Git 会创建一个标签对象，并记录一个引用来指向该标签对象，而不是直接指向提交对象。 可以通过创建一个附注标签来验证这个过程（使用`-a`选项）：

```bash
$ git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'test tag'
```

下面是上述过程所建标签对象的 SHA-1 值：

```bash
$ cat .git/refs/tags/v1.1
9585191f37f7b0fb9444f35a9bf50de191beadc2
```

现在对该 SHA-1 值运行`git cat-file -p`命令：

```bash
$ git cat-file -p 9585191f37f7b0fb9444f35a9bf50de191beadc2
object 1a410efbd13591db07496601ebc7a059dd55cfe9
type commit
tag v1.1
tagger Scott Chacon <schacon@gmail.com> Sat May 23 16:48:58 2009 -0700

test tag
```

我们注意到，object 条目指向我们打了标签的那个提交对象的 SHA-1 值。 另外要注意的是，标签对象并非必须指向某个提交对象；你可以对任意类型的 Git 对象打标签。 例如，在 Git 源码中，项目维护者将他们的 GPG 公钥添加为一个数据对象，然后对这个对象打了一个标签。 可以克隆一个 Git 版本库，然后通过执行下面的命令来在这个版本库中查看上述公钥：

```bash
$ git cat-file blob junio-gpg-pub
```

Linux 内核版本库同样有一个不指向提交对象的标签对象——首个被创建的标签对象所指向的是最初被引入版本库的那份内核源码所对应的树对象。

### 3.3 远程引用

我们将看到的第三种引用类型是远程引用（remote reference）。 如果你添加了一个远程版本库并对其执行过推送操作，Git 会记录下最近一次推送操作时每一个分支所对应的值，并保存在 refs/remotes 目录下。 例如，你可以添加一个叫做 origin 的远程版本库，然后把 master 分支推送上去：

```bash
$ git remote add origin git@github.com:schacon/simplegit-progit.git
$ git push origin master
Counting objects: 11, done.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (7/7), 716 bytes, done.
Total 7 (delta 2), reused 4 (delta 1)
To git@github.com:schacon/simplegit-progit.git
  a11bef0..ca82a6d  master -> master
```

此时，如果查看 refs/remotes/origin/master 文件，可以发现 origin 远程版本库的 master 分支所对应的 SHA-1 值，就是最近一次与服务器通信时本地 master 分支所对应的 SHA-1 值：

```bash
$ cat .git/refs/remotes/origin/master
ca82a6dff817ec66f44342007202690a93763949
```

远程引用和分支（位于 refs/heads 目录下的引用）之间最主要的区别在于，远程引用是只读的。 虽然可以`git checkout`到某个远程引用，但是 Git 并不会将 HEAD 引用指向该远程引用。因此，你永远不能通过`commit`命令来更新远程引用。 Git 将这些远程引用作为记录远程服务器上各分支最后已知位置状态的书签来管理。

## 4.包文件

如果你跟着做完了上一节中的所有步骤，那么现在应该有了一个测试用的 Git 仓库， 其中包含 11 个对象：四个数据对象，三个树对象，三个提交对象和一个标签对象：

```bash
$ find .git/objects -type f
.git/objects/01/55eb4229851634a0f03eb265b69f5a2d56f341 # tree 2
.git/objects/1a/410efbd13591db07496601ebc7a059dd55cfe9 # commit 3
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a # test.txt v2
.git/objects/3c/4e9cd789d88d8d89c1073707c3585e41b0e614 # tree 3
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30 # test.txt v1
.git/objects/95/85191f37f7b0fb9444f35a9bf50de191beadc2 # tag
.git/objects/ca/c0cab538b970a37ea1e769cbbde608743bc96d # commit 2
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4 # 'test content'
.git/objects/d8/329fc1cc938780ffdd9f94e0d364e0ea74f579 # tree 1
.git/objects/fa/49b077972391ad58037050f2a75f74e3671e92 # new.txt
.git/objects/fd/f4fc3344e67ab068f836878b6c4951e3b15f3d # commit 1
```

Git 使用 zlib 压缩这些文件的内容，而且我们并没有存储太多东西，所以上文中的文件一共只占用了 925 字节。接下来，我们会指引你添加一些大文件到仓库中，以此展示 Git 的一个很有趣的功能。 为了便于演示，我们要把之前在 Grit 库中用到过的 repo.rb 文件添加进来——这是一个大小约为 22K 的源代码文件：

```bash
$ curl
https://raw.githubusercontent.com/mojombo/grit/master/lib/grit/repo.rb >
repo.rb
$ git checkout master
$ git add repo.rb
$ git commit -m 'added repo.rb'
[master 484a592] added repo.rb
 3 files changed, 709 insertions(+), 2 deletions(-)
 delete mode 100644 bak/test.txt
 create mode 100644 repo.rb
 rewrite test.txt (100%)
```

如果你查看生成的树对象，可以看到 repo.rb 文件对应的数据对象的 SHA-1 值：

```bash
$ git cat-file -p master^{tree}
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob 033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5      repo.rb
100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt
```

接下来你可以使用`git cat-file`命令查看这个对象有多大：

```bash
$ git cat-file -s 033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5
22044
```

现在，稍微修改这个文件，然后看看会发生什么：

```bash
$ echo '# testing' >> repo.rb
$ git commit -am 'modified repo.rb a bit'
[master 2431da6] modified repo.rb a bit
 1 file changed, 1 insertion(+)
```

查看这个最新的提交生成的树对象，你会看到一些有趣的东西：

```bash
$ git cat-file -p master^{tree}
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob b042a60ef7dff760008df33cee372b945b6e884e      repo.rb
100644 blob e3f094f522629ae358806b17daf78246c27c007b      test.txt
```

repo.rb 对应一个与之前完全不同的数据对象，这意味着，虽然你只是在一个 400 行的文件后面加入一行新内容，Git 也会用一个全新的对象来存储新的文件内容：

```bash
$ git cat-file -s b042a60ef7dff760008df33cee372b945b6e884e
22054
```

你的磁盘上现在有两个几乎完全相同、大小均为 22K 的对象（每个都被压缩到大约 7K）。 如果 Git 只完整保存其中一个，再保存另一个对象与之前版本的差异内容，岂不更好？

事实上 Git 可以那样做。 Git 最初向磁盘中存储对象时所使用的格式被称为“松散（loose）”对象格式。 但是，Git 会时不时地将多个这些对象打包成一个称为“包文件（packfile）”的二进制文件，以节省空间和提高效率。 当版本库中有太多的松散对象，或者你手动执行`git gc`命令，或者你向远程服务器执行推送时，Git 都会这样做。 要看到打包过程，你可以手动执行`git gc`命令让 Git 对对象进行打包：

```bash
$ git gc
Counting objects: 18, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (14/14), done.
Writing objects: 100% (18/18), done.
Total 18 (delta 3), reused 0 (delta 0)
```

这个时候再查看 objects 目录，你会发现大部分的对象都不见了，与此同时出现了一对新文件：

```bash
$ find .git/objects -type f
.git/objects/bd/9dbf5aae1a3862dd1526723246b20206e5fc37
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
.git/objects/info/packs
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.idx
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.pack
```

仍保留着的几个对象是未被任何提交记录引用的数据对象——在此例中是你之前创建的 “what is up, doc?” 和 “test content” 这两个示例数据对象。 因为你从没将它们添加至任何提交记录中，所以 Git 认为它们是悬空（dangling）的，不会将它们打包进新生成的包文件中。

剩下的文件是新创建的包文件和一个索引。 包文件包含了刚才从文件系统中移除的所有对象的内容。 索引文件包含了包文件的偏移信息，我们通过索引文件就可以快速定位任意一个指定对象。 有意思的是运行`gc`命令前磁盘上的对象大小约为 15K，而这个新生成的包文件大小仅有 7K。 通过打包对象减少了一半的磁盘占用空间。

Git 是如何做到这点的？ Git 打包对象时，会查找命名及大小相近的文件，并只保存文件不同版本之间的差异内容。 你可以查看包文件，观察它是如何节省空间的。`git verify-pack`这个底层命令可以让你查看已打包的内容：

```bash
$ git verify-pack -v .git/objects/pack/pack-
978e03944f5c581011e6998cd0e9e30000905586.idx
2431da676938450a4d72e260db3bf7b0f587bbc1 commit 223 155 12
69bcdaff5328278ab1c0812ce0e07fa7d26a96d7 commit 214 152 167
80d02664cb23ed55b226516648c7ad5d0a3deb90 commit 214 145 319
43168a18b7613d1281e5560855a83eb8fde3d687 commit 213 146 464
092917823486a802e94d727c820a9024e14a1fc2 commit 214 146 610
702470739ce72005e2edff522fde85d52a65df9b commit 165 118 756
d368d0ac0678cbe6cce505be58126d3526706e54 tag    130 122 874
fe879577cb8cffcdf25441725141e310dd7d239b tree   136 136 996
d8329fc1cc938780ffdd9f94e0d364e0ea74f579 tree   36 46 1132
deef2e1b793907545e50a2ea2ddb5ba6c58c4506 tree   136 136 1178
d982c7cb2c2a972ee391a85da481fc1f9127a01d tree   6 17 1314 1 \
  deef2e1b793907545e50a2ea2ddb5ba6c58c4506
3c4e9cd789d88d8d89c1073707c3585e41b0e614 tree   8 19 1331 1 \
  deef2e1b793907545e50a2ea2ddb5ba6c58c4506
0155eb4229851634a0f03eb265b69f5a2d56f341 tree   71 76 1350
83baae61804e65cc73a7201a7252750c76066a30 blob   10 19 1426
fa49b077972391ad58037050f2a75f74e3671e92 blob   9 18 1445
b042a60ef7dff760008df33cee372b945b6e884e blob   22054 5799 1463
033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5 blob   9 20 7262 1 \
  b042a60ef7dff760008df33cee372b945b6e884e
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a blob   10 19 7282
non delta: 15 objects
chain length = 1: 3 objects
.git/objects/pack/pack-978e03944f5c581011e6998cd0e9e30000905586.pack: ok
```

此处，033b4 这个数据对象（即 repo.rb 文件的第一个版本，如果你还记得的话）引用了数据对象 b042a，即该文件的第二个版本。 命令输出内容的第三列显示的是各个对象在包文件中的大小，可以看到 b042a 占用了 22K 空间，而 033b4 仅占用 9 字节。 同样有趣的地方在于，第二个版本完整保存了文件内容，而原始的版本反而是以差异方式保存的——这是因为大部分情况下需要快速访问文件的最新版本。

最妙之处是你可以随时重新打包。 Git 时常会自动对仓库进行重新打包以节省空间。当然你也可以随时手动执行`git gc`命令来这么做。

## 5.引用规范

纵观全书，我们已经使用过一些诸如远程分支到本地引用的简单映射方式，但这种映射可以更复杂。 假设你已经跟着前几节在本地创建了一个小的 Git 仓库，现在想要添加一个远程仓库：

```bash
$ git remote add origin https://github.com/schacon/simplegit-progit
```

运行上述命令会在你仓库中的 .git/config 文件中添加一个小节， 并在其中指定远程版本库的名称（origin）、URL 和一个用于获取操作的 引用规范（refspec）：

```bash
[remote "origin"]
    url = https://github.com/schacon/simplegit-progit
    fetch = +refs/heads/*:refs/remotes/origin/*
```

引用规范的格式由一个可选的`+`号和紧随其后的`<src>:<dst>`组成， 其中`<src>`是一个模式（pattern），代表远程版本库中的引用；`<dst>`是本地跟踪的远程引用的位置。`+`号告诉 Git 即使在不能快进的情况下也要（强制）更新引用。

默认情况下，引用规范由`git remote add origin`命令自动生成， Git 获取服务器中 refs/heads/ 下面的所有引用，并将它写入到本地的 refs/remotes/origin/ 中。 所以，如果服务器上有一个 master 分支，你可以在本地通过下面任意一种方式来访问该分支上的提交记录：

```bash
$ git log origin/master
$ git log remotes/origin/master
$ git log refs/remotes/origin/master
```

上面的三个命令作用相同，因为 Git 会把它们都扩展成`refs/remotes/origin/master`。

如果想让 Git 每次只拉取远程的 master 分支，而不是所有分支， 可以把（引用规范的）获取那一行修改为只引用该分支：

```bash
fetch = +refs/heads/master:refs/remotes/origin/master
```

这仅是针对该远程版本库的`git fetch`操作的默认引用规范。 如果有某些只希望被执行一次的操作，我们也可以在命令行指定引用规范。 若要将远程的 master 分支拉到本地的 origin/mymaster 分支，可以运行：

```bash
$ git fetch origin master:refs/remotes/origin/mymaster
```

你也可以指定多个引用规范。 在命令行中，你可以按照如下的方式拉取多个分支：

```bash
$ git fetch origin master:refs/remotes/origin/mymaster topic:refs/remotes/origin/topic
From git@github.com:schacon/simplegit
 ! [rejected]        master     -> origin/mymaster  (non fast forward)
 * [new branch]      topic      -> origin/topic
```

在这个例子中，对 master 分支的拉取操作被拒绝，因为它不是一个可以快进的引用。 我们可以通过在引用规范之前指定`+`号来覆盖该规则。

你也可以在配置文件中指定多个用于获取操作的引用规范。 如果想在每次从 origin 远程仓库获取时都包括 master 和 experiment 分支，添加如下两行：

```bash
[remote "origin"]
    url = https://github.com/schacon/simplegit-progit
    fetch = +refs/heads/master:refs/remotes/origin/master
    fetch = +refs/heads/experiment:refs/remotes/origin/experiment
```

我们不能在模式中使用部分通配符，所以像下面这样的引用规范是不合法的：

```bash
fetch = +refs/heads/qa*:refs/remotes/origin/qa*
```

但我们可以使用命名空间（或目录）来达到类似目的。 假设你有一个 QA 团队，他们推送了一系列分支，同时你只想要获取 master 和 QA 团队的所有分支而不关心其他任何分支，那么可以使用如下配置：

```bash
[remote "origin"]
    url = https://github.com/schacon/simplegit-progit
    fetch = +refs/heads/master:refs/remotes/origin/master
    fetch = +refs/heads/qa/*:refs/remotes/origin/qa/*
```

如果项目的工作流很复杂，有 QA 团队推送分支、开发人员推送分支、集成团队推送并且在远程分支上展开协作，你就可以像这样（在本地）为这些分支创建各自的命名空间，非常方便。

### 5.1 引用规范推送

像上面这样从远程版本库获取已在命名空间中的引用当然很棒，但 QA 团队最初应该如何将他们的分支放入远程的 qa/ 命名空间呢？ 我们可以通过引用规范推送来完成这个任务。

如果 QA 团队想把他们的 master 分支推送到远程服务器的 qa/master 分支上，可以运行：

```bash
$ git push origin master:refs/heads/qa/master
```

如果他们希望 Git 每次运行`git push origin`时都像上面这样推送，可以在他们的配置文件中添加一条 push 值：

```bash
[remote "origin"]
    url = https://github.com/schacon/simplegit-progit
    fetch = +refs/heads/*:refs/remotes/origin/*
    push = refs/heads/master:refs/heads/qa/master
```

正如刚才所指出的，这会让`git push origin`默认把本地 master 分支推送到远程 qa/master 分支。

### 5.2 删除引用

你还可以借助类似下面的命令通过引用规范从远程服务器上删除引用：

```bash
$ git push origin :topic
```

因为引用规范（的格式）是`<src>:<dst>`，所以上述命令把`<src>`留空，意味着把远程版本库的 topic 分支定义为空值，也就是删除它。

或者你可以使用更新的语法（自 Git v1.7.0 以后可用）：

```bash
$ git push origin --delete topic
```

## 6.传输协议

Git 可以通过两种主要的方式在版本库之间传输数据：“哑（dumb）”协议和“智能（smart）”协议。

### 6.1 哑协议

如果你正在架设一个基于 HTTP 协议的只读版本库，一般而言这种情况下使用的就是哑协议。 这个协议之所以被称为“哑”协议，是因为在传输过程中，服务端不需要有针对 Git 特有的代码；抓取过程是一系列 HTTP 的 GET 请求，这种情况下，客户端可以推断出服务端 Git 仓库的布局。

> 现在已经很少使用哑协议了。 使用哑协议的版本库很难保证安全性和私有化，所以大多数 Git服务器宿主（包括云端和本地）都会拒绝使用它。 一般情况下都建议使用智能协议，我们会在后面进行介绍。

让我们通过 simplegit 版本库来看看`http-fetch`的过程：

```bash
$ git clone http://server/simplegit-progit.git
```

它做的第一件事就是拉取 info/refs 文件。 这个文件是通过`update-server-info`命令生成的，这也解释了在使用 HTTP 传输时，必须把它设置为 post-receive 钩子的原因：

```bash
=> GET info/refs
ca82a6dff817ec66f44342007202690a93763949     refs/heads/master
```

现在，你得到了一个远程引用和 SHA-1 值的列表。 接下来，你要确定 HEAD 引用是什么，这样你就知道在完成后应该被检出到工作目录的内容：

```bash
=> GET HEAD
ref: refs/heads/master
```

这说明在完成抓取后，你需要检出 master 分支。 这时，你就可以开始遍历处理了。 因为你是从 info/refs 文件中所提到的 ca82a6 提交对象开始的，所以你的首要操作是获取它：

```bash
=> GET objects/ca/82a6dff817ec66f44342007202690a93763949
(179 bytes of binary data)
```

你取回了一个对象——这是一个在服务端以松散格式保存的对象，是你通过使用静态 HTTP GET 请求获取的。你可以使用 zlib 解压缩它，去除其头部，查看提交记录的内容：

```bash
$ git cat-file -p ca82a6dff817ec66f44342007202690a93763949
tree cfda3bf379e4f8dba8717dee55aab78aef7f4daf
parent 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
author Scott Chacon <schacon@gmail.com> 1205815931 -0700
committer Scott Chacon <schacon@gmail.com> 1240030591 -0700

changed the version number
```

接下来，你还要再获取两个对象，一个是树对象 cfda3b，它包含有我们刚刚获取的提交对象所指向的内容，另一个是它的父提交 085bb3：

```bash
=> GET objects/08/5bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
(179 bytes of data)
```

这样就取得了你的下一个提交对象。 再抓取树对象：

```bash
=> GET objects/cf/da3bf379e4f8dba8717dee55aab78aef7f4daf
(404 - Not Found)
```

噢——看起来这个树对象在服务端并不以松散格式对象存在，所以你得到了一个 404 响应，代表在 HTTP 服务端没有找到该对象。 这有好几个可能的原因——这个对象可能在替代版本库里面，或者在包文件里面。 Git 会首先检查所有列出的替代版本库：

```bash
=> GET objects/info/http-alternates
(empty file)
```

如果这返回了一个包含替代版本库 URL 的列表，那么 Git 就会去那些地址检查松散格式对象和文件——这是一种能让派生项目共享对象以节省磁盘的好方法。 然而，在这个例子中，没有列出可用的替代版本库。所以你所需要的对象肯定在某个包文件中。 要检查服务端有哪些可用的包文件，你需要获取 objects/info/packs 文件，这里面有一个包文件列表（它也是通过执行 update-server-info 所生成的）：

```bash
=> GET objects/info/packs
P pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack
```

服务端只有一个包文件，所以你要的对象显然就在里面。但是你要先检查它的索引文件以确认。 即使服务端有多个包文件，这也是很有用的，因为这样你就可以知道你所需要的对象是在哪一个包文件里面：

```bash
=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.idx
(4k of binary data)
```

现在你有这个包文件的索引，你可以查看你要的对象是否在里面—— 因为索引文件列出了这个包文件所包含的所有对象的 SHA-1 值，和该对象存在于包文件中的偏移量。 你的对象就在这里，接下来就是获取整个包文件：

```bash
=> GET objects/pack/pack-816a9b2334da9953e530f27bcac22082a9f5b835.pack
(13k of binary data)
```

现在你也有了你的树对象，你可以继续在提交记录上漫游。 它们全部都在这个你刚下载的包文件里面，所以你不用继续向服务端请求更多下载了。 Git 会将开始时下载的 HEAD 引用所指向的 master 分支检出到工作目录。

### 6.2 智能协议

哑协议虽然很简单但效率略低，且它不能从客户端向服务端发送数据。 智能协议是更常用的传送数据的方法，但它需要在服务端运行一个进程，而这也是 Git 的智能之处——它可以读取本地数据，理解客户端有什么和需要什么，并为它生成合适的包文件。 总共有两组进程用于传输数据，它们分别负责上传和下载数据。

#### 1.上传数据

为了上传数据至远端，Git 使用`send-pack`和`receive-pack`进程。 运行在客户端上的`send-pack`进程连接到远端运行的`receive-pack`进程。

##### 1.1 SSH

举例来说，在项目中使用命令`git push origin master`时，origin 是由基于 SSH 协议的 URL 所定义的。 Git 会运行`send-pack`进程，它会通过 SSH 连接你的服务器。 它会尝试通过 SSH 在服务端执行命令，就像这样：

```bash
$ ssh -x git@server "git-receive-pack 'simplegit-progit.git'"
00a5ca82a6dff817ec66f4437202690a93763949 refs/heads/master report-status \
    delete-refs side-band-64k quiet ofs-delta \
    agent=git/2:2.1.1+github-607-gfba4028 delete-refs
0000
```

`git-receive-pack`命令会立即为它所拥有的每一个引用发送一行响应——在这个例子中，就只有 master 分支和它的 SHA-1 值。 第一行响应中也包含了一个服务端能力的列表（这里是`report-status`、`delete-refs`和一些其它的，包括客户端的识别码）。

每一行以一个四位的十六进制值开始，用于指明本行的长度。 你看到第一行以 00a5 开始，这在十六进制中表示 165，意味着第一行有 165 字节。 下一行是 0000，表示服务端已完成了发送引用列表过程。

现在它知道了服务端的状态，你的 send-pack 进程会判断哪些提交记录是它所拥有但服务端没有的。 send-pack 会告知 receive-pack 这次推送将会更新的各个引用。 举个例子，如果你正在更新 master 分支，并且增加 experiment 分支，这个 send-pack 的响应将会是像这样：

```bash
0076ca82a6dff817ec66f44342007202690a93763949
15027957951b64cf874c3557a0f3547bd83b3ff6 \
    refs/heads/master report-status
006c0000000000000000000000000000000000000000
cdfdb42577e2506715f8cfeacdbabc092bf63e8d \
    refs/heads/experiment
0000
```

第一行也包括了客户端的能力。 这里的全为 0 的 SHA-1 值表示之前没有过这个引用——因为你正要添加新的 experiment 引用。 删除引用时，将会看到相反的情况：右边的 SHA-1 值全为 0。

然后，客户端会发送一个包含了所有服务端上所没有的对象的包文件。 最终，服务端会响应一个成功（或失败）的标识。

```bash
000eunpack ok
```

##### 1.2 HTTP(S)

上传过程在 HTTP 上几乎是相同的，除了握手阶段有一点小区别。 连接是从下面这个请求开始的：

```bash
=> GET http://server/simplegit-progit.git/info/refs?service=git-receive-
pack
001f# service=git-receive-pack
00ab6c5f0e45abd7832bf23074a333f739977c9e8188 refs/heads/master report-
status \
    delete-refs side-band-64k quiet ofs-delta \
    agent=git/2:2.1.1~vmg-bitmaps-bugaloo-608-g116744e
0000
```

这完成了客户端和服务端的第一次数据交换。 接下来客户端发起另一个请求，这次是一个 POST 请求，这个请求中包含了 send-pack 提供的数据。

```bash
=> POST http://server/simplegit-progit.git/git-receive-pack
```

这个 POST 请求的内容是 send-pack 的输出和相应的包文件。 服务端在收到请求后相应地作出成功或失败的 HTTP 响应。

请牢记，HTTP 协议有可能会进一步用分块传输编码将数据包裹起来。

#### 2.下载数据

当你在下载数据时，fetch-pack 和 upload-pack 进程就起作用了。 客户端启动 fetch-pack 进程，连接至远端的 upload-pack 进程，以协商后续传输的数据。

##### 2.1 SSH

如果你通过 SSH 使用抓取功能，fetch-pack 会像这样运行：

```bash
$ ssh -x git@server "git-upload-pack 'simplegit-progit.git'"
```

在 fetch-pack 连接后，upload-pack 会返回类似下面的内容：

```bash
00dfca82a6dff817ec66f44342007202690a93763949 HEAD multi_ack thin-pack \
    side-band side-band-64k ofs-delta shallow no-progress include-tag \
    multi_ack_detailed symref=HEAD:refs/heads/master \
    agent=git/2:2.1.1+github-607-gfba4028
003fe2409a098dc3e53539a9028a94b6224db9d6a6b6 refs/heads/master
0000
```

这与 receive-pack 的响应很相似，但是这里所包含的能力是不同的。 而且它还包含 HEAD 引用所指向内容（symref=HEAD:refs/heads/master），这样如果客户端执行的是克隆，它就会知道要检出什么。

这时候，fetch-pack 进程查看它自己所拥有的对象，并响应 “want” 和它需要的对象的 SHA-1 值。 它还会发送“have”和所有它已拥有的对象的 SHA-1 值。 在列表的最后，它还会发送 “done” 以通知 upload-pack 进程可以开始发送它所需对象的包文件：

```bash
003cwant ca82a6dff817ec66f44342007202690a93763949 ofs-delta
0032have 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
0009done
0000
```

##### 2.2 HTTP(S)

抓取操作的握手需要两个 HTTP 请求。 第一个是向和哑协议中相同的端点发送 GET 请求：

```bash
=> GET $GIT_URL/info/refs?service=git-upload-pack
001e# service=git-upload-pack
00e7ca82a6dff817ec66f44342007202690a93763949 HEAD multi_ack thin-pack \
    side-band side-band-64k ofs-delta shallow no-progress include-tag \
    multi_ack_detailed no-done symref=HEAD:refs/heads/master \
    agent=git/2:2.1.1+github-607-gfba4028
003fca82a6dff817ec66f44342007202690a93763949 refs/heads/master
0000
```

这和通过 SSH 使用 git-upload-pack 是非常相似的，但是第二个数据交换则是一个单独的请求：

```bash
=> POST $GIT_URL/git-upload-pack HTTP/1.0
0032want 0a53e9ddeaddad63ad106860237bbf53411d11a7
0032have 441b40d833fdfa93eb2908e52742248faf0ee993
0000
```

这个输出格式还是和前面一样的。 这个请求的响应包含了所需要的包文件，并指明成功或失败。

## 7.维护与数据恢复

有的时候，你需要对仓库进行清理——使它的结构变得更紧凑，或是对导入的仓库进行清理，或是恢复丢失的内容。 这个小节将会介绍这些情况中的一部分。

### 7.1 维护

Git 会不定时地自动运行一个叫做 “auto gc” 的命令。 大多数时候，这个命令并不会产生效果。 然而，如果有太多松散对象（不在包文件中的对象）或者太多包文件，Git 会运行一个完整的 git gc 命令。 “gc” 代表垃圾回收，这个命令会做以下事情：收集所有松散对象并将它们放置到包文件中， 将多个包文件合并为一个大的包文件，移除与任何提交都不相关的陈旧对象。

可以像下面一样手动执行自动垃圾回收：

```bash
$ git gc --auto
```

就像上面提到的，这个命令通常并不会产生效果。 大约需要 7000 个以上的松散对象或超过 50 个的包文件才能让 Git 启动一次真正的 gc 命令。 你可以通过修改`gc.auto`与`gc.autopacklimit`的设置来改动这些数值。

gc 将会做的另一件事是打包你的引用到一个单独的文件。 假设你的仓库包含以下分支与标签：

```bash
$ find .git/refs -type f
.git/refs/heads/experiment
.git/refs/heads/master
.git/refs/tags/v1.0
.git/refs/tags/v1.1
```

如果你执行了`git gc`命令，refs 目录中将不会再有这些文件。 为了保证效率 Git 会将它们移动到名为 .git/packed-refs 的文件中，就像这样：

```bash
$ cat .git/packed-refs
# pack-refs with: peeled fully-peeled
cac0cab538b970a37ea1e769cbbde608743bc96d refs/heads/experiment
ab1afef80fac8e34258ff41fc1b867c702daa24b refs/heads/master
cac0cab538b970a37ea1e769cbbde608743bc96d refs/tags/v1.0
9585191f37f7b0fb9444f35a9bf50de191beadc2 refs/tags/v1.1
^1a410efbd13591db07496601ebc7a059dd55cfe9
```

如果你更新了引用，Git 并不会修改这个文件，而是向 refs/heads 创建一个新的文件。 为了获得指定引用的正确 SHA-1 值，Git 会首先在 refs 目录中查找指定的引用，然后再到 packed-refs 文件中查找。 所以，如果你在 refs 目录中找不到一个引用，那么它或许在 packed-refs 文件中。

注意这个文件的最后一行，它会以`^`开头。 这个符号表示它上一行的标签是附注标签，`^`所在的那一行是附注标签指向的那个提交。

### 7.2 数据恢复

在你使用 Git 的时候，你可能会意外丢失一次提交。 通常这是因为你强制删除了正在工作的分支，但是最后却发现你还需要这个分支， 亦或者硬重置了一个分支，放弃了你想要的提交。 如果这些事情已经发生，该如何找回你的提交呢？

下面的例子将硬重置你的测试仓库中的 master 分支到一个旧的提交，以此来恢复丢失的提交。 首先，让我们看看你的仓库现在在什么地方：

```bash
$ git log --pretty=oneline
ab1afef80fac8e34258ff41fc1b867c702daa24b modified repo a bit
484a59275031909e19aadb7c92262719cfcdf19a added repo.rb
1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
cac0cab538b970a37ea1e769cbbde608743bc96d second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit
```

现在，我们将 master 分支硬重置到第三次提交：

```bash
$ git reset --hard 1a410efbd13591db07496601ebc7a059dd55cfe9
HEAD is now at 1a410ef third commit
$ git log --pretty=oneline
1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
cac0cab538b970a37ea1e769cbbde608743bc96d second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit
```

现在顶部的两个提交已经丢失了——没有分支指向这些提交。 你需要找出最后一次提交的 SHA-1 然后增加一个指向它的分支。 窍门就是找到最后一次的提交的 SHA-1 ——但是估计你记不起来了，对吗？

最方便，也是最常用的方法，是使用一个名叫`git reflog`的工具。 当你正在工作时，Git 会默默地记录每一次你改变 HEAD 时它的值。 每一次你提交或改变分支，引用日志都会被更新。 引用日志（reflog）也可以通过`git update-ref`命令更新，我们在 Git 引用 有提到使用这个命令而不是是直接将 SHA-1 的值写入引用文件中的原因。 你可以在任何时候通过执行`git reflog`命令来了解你曾经做过什么：

```bash
$ git reflog
1a410ef HEAD@{0}: reset: moving to 1a410ef
ab1afef HEAD@{1}: commit: modified repo.rb a bit
484a592 HEAD@{2}: commit: added repo.rb
```

这里可以看到我们已经检出的两次提交，然而并没有足够多的信息。 为了使显示的信息更加有用，我们可以执行`git log -g`，这个命令会以标准日志的格式输出引用日志。

```bash
$ git log -g
commit 1a410efbd13591db07496601ebc7a059dd55cfe9
Reflog: HEAD@{0} (Scott Chacon <schacon@gmail.com>)
Reflog message: updating HEAD
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:22:37 2009 -0700

        third commit

commit ab1afef80fac8e34258ff41fc1b867c702daa24b
Reflog: HEAD@{1} (Scott Chacon <schacon@gmail.com>)
Reflog message: updating HEAD
Author: Scott Chacon <schacon@gmail.com>
Date:   Fri May 22 18:15:24 2009 -0700

       modified repo.rb a bit
```

看起来下面的那个就是你丢失的提交，你可以通过创建一个新的分支指向这个提交来恢复它。 例如，你可以创建一个名为 recover-branch 的分支指向这个提交（ab1afef）：

```bash
$ git branch recover-branch ab1afef
$ git log --pretty=oneline recover-branch
ab1afef80fac8e34258ff41fc1b867c702daa24b modified repo a bit
484a59275031909e19aadb7c92262719cfcdf19a added repo.rb
1a410efbd13591db07496601ebc7a059dd55cfe9 third commit
cac0cab538b970a37ea1e769cbbde608743bc96d second commit
fdf4fc3344e67ab068f836878b6c4951e3b15f3d first commit
```

不错，现在有一个名为 recover-branch 的分支是你的 master 分支曾经指向的地方， 再一次使得前两次提交可到达了。接下来，假设你丢失的提交因为某些原因不在引用日志中， 那么我们可以通过移除 recover-branch 分支并删除引用日志来模拟这种情况。 现在前两次提交又不被任何分支指向了：

```bash
$ git branch -D recover-branch
$ rm -Rf .git/logs/
```

由于引用日志数据存放在 .git/logs/ 目录中，现在你已经没有引用日志了。 这时该如何恢复那次提交？ 一种方式是使用`git fsck`实用工具，将会检查数据库的完整性。 如果使用一个`--full`选项运行它，它会向你显示出所有没有被其他对象指向的对象：

```bash
$ git fsck --full
Checking object directories: 100% (256/256), done.
Checking objects: 100% (18/18), done.
dangling blob d670460b4b4aece5915caf5c68d12f560a9fe3e4
dangling commit ab1afef80fac8e34258ff41fc1b867c702daa24b
dangling tree aea790b9a58f6cf6f2804eeac9f0abbe9631e4c9
dangling blob 7108f7ecb345ee9d0084193f147cdad4d2998293
```

在这个例子中，你可以在 “dangling commit” 后看到你丢失的提交。 现在你可以用和之前相同的方法恢复这个提交，也就是添加一个指向这个提交的分支。

### 7.3 移除对象

Git 有很多很棒的功能，但是其中一个特性会导致问题，`git clone`会下载整个项目的历史，包括每一个文件的每一个版本。 如果所有的东西都是源代码那么这很好，因为 Git 被高度优化来有效地存储这种数据。 然而，如果某个人在之前向项目添加了一个大小特别大的文件，即使你将这个文件从项目中移除了，每次克隆还是都要强制的下载这个大文件。 之所以会产生这个问题，是因为这个文件在历史中是存在的，它会永远在那里。

==警告：这个操作对提交历史的修改是破坏性的==。它会从你必须修改或移除一个大文件引用最早的树对象开始重写每一次提交。 如果你在导入仓库后，在任何人开始基于这些提交工作前执行这个操作，那么将不会有任何问题——否则， 你必须通知所有的贡献者他们需要将他们的成果变基到你的新提交上。

为了演示，我们将添加一个大文件到测试仓库中，并在下一次提交中删除它，现在我们需要找到它，并将它从仓库中永久删除。 首先，添加一个大文件到仓库中：

```bash
$ curl https://www.kernel.org/pub/software/scm/git/git-2.1.0.tar.gz > git.tgz
$ git add git.tgz
$ git commit -m 'add git tarball'
[master 7b30847] add git tarball
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 git.tgz
```

哎呀——其实这个项目并不需要这个巨大的压缩文件。 现在我们将它移除：

```bash
$ git rm git.tgz
rm 'git.tgz'
$ git commit -m 'oops - removed large tarball'
[master dadf725] oops - removed large tarball
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 git.tgz
```

现在，我们执行 gc 来查看数据库占用了多少空间：

```bash
$ git gc
Counting objects: 17, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (13/13), done.
Writing objects: 100% (17/17), done.
Total 17 (delta 1), reused 10 (delta 0)
```

你也可以执行`count-objects`命令来快速的查看占用空间大小：

```bash
$ git count-objects -v
count: 7
size: 32
in-pack: 17
packs: 1
size-pack: 4868
prune-packable: 0
garbage: 0
size-garbage: 0
```

size-pack 的数值指的是你的包文件以 KB 为单位计算的大小，所以你大约占用了 5MB 的空间。 在最后一次提交前，使用了不到 2KB ——显然，从之前的提交中移除文件并不能从历史中移除它。 每一次有人克隆这个仓库时，他们将必须克隆所有的 5MB 来获得这个微型项目，只因为你意外地添加了一个大文件。

现在来让我们彻底的移除这个文件。

首先你必须找到它。 在本例中，你已经知道是哪个文件了。 但是假设你不知道；该如何找出哪个文件或哪些文件占用了如此多的空间？ 如果你执行`git gc`命令，所有的对象将被放入一个包文件中，你可以通过运行`git verify-pack`命令， 然后对输出内容的第三列（即文件大小）进行排序，从而找出这个大文件。 你也可以将这个命令的执行结果通过管道传送给`tail`命令，因为你只需要找到列在最后的几个大对象。

```bash
$ git verify-pack -v .git/objects/pack/pack-29…69.idx | sort -k 3 -n | tail -3
dadf7258d699da2c8d89b09ef6670edb7d5f91b4 commit 229 159 12
033b4468fa6b2a9547a70d88d1bbe8bf3f9ed0d5 blob   22044 5792 4977696
82c99a3e86bb1267b236a4b6eff7868d97489af1 blob   4975916 4976258 1438
```

你可以看到这个大对象出现在返回结果的最底部：占用 5MB 空间。 为了找出具体是哪个文件，可以使用`rev-list`命令，我们在【指定特殊的提交信息格式】中曾提到过。 如果你传递`--objects`参数给`rev-list`命令，它就会列出所有提交的 SHA-1、数据对象的 SHA-1 和与它们相关联的文件路径。 可以使用以下命令来找出你的数据对象的名字：

```bash
$ git rev-list --objects --all | grep 82c99a3
82c99a3e86bb1267b236a4b6eff7868d97489af1 git.tgz
```

现在，你只需要从过去所有的树中移除这个文件。 使用以下命令可以轻松地查看哪些提交对这个文件产生改动：

```bash
$ git log --oneline --branches -- git.tgz
dadf725 oops - removed large tarball
7b30847 add git tarball
```

现在，你必须重写 7b30847 提交之后的所有提交来从 Git 历史中完全移除这个文件。 为了执行这个操作，我们要使用`filter-branch`命令：

```bash
$ git filter-branch --index-filter 'git rm --ignore-unmatch --cached git.tgz' -- 7b30847^..
Rewrite 7b30847d080183a1ab7d18fb202473b3096e9f34 (1/2)rm 'git.tgz'
Rewrite dadf7258d699da2c8d89b09ef6670edb7d5f91b4 (2/2)
Ref 'refs/heads/master' was rewritten
```

`--index-filter`选项并不会让命令将修改在硬盘上检出的文件，而只是修改在暂存区或索引中的文件。

你必须使用`git rm --cached`命令来移除文件，而不是通过类似`rm file`的命令——因为你需要从索引中移除它，而不是磁盘中。 还有一个原因是速度—— Git 在运行过滤器时，并不会检出每个修订版本到磁盘中，所以这个过程会非常快。 如果愿意的话，你也可以通过`--tree-filter`选项来完成同样的任务。`git rm`命令的`--ignore-unmatch`选项告诉命令：如果尝试删除的模式不存在时，不提示错误。最后，使用`filter-branch`选项来重写自 7b30847 提交以来的历史，也就是这个问题产生的地方。 否则，这个命令会从最旧的提交开始，这将会花费许多不必要的时间。

你的历史中将不再包含对那个文件的引用。 不过，你的引用日志和你在 .git/refs/original 通过 filter-branch 选项添加的新引用中还存有对这个文件的引用，所以你必须移除它们然后重新打包数据库。 在重新打包前需要移除任何包含指向那些旧提交的指针的文件：

```bash
$ rm -Rf .git/refs/original
$ rm -Rf .git/logs/
$ git gc
Counting objects: 15, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (11/11), done.
Writing objects: 100% (15/15), done.
Total 15 (delta 1), reused 12 (delta 0)
```

让我们看看你省了多少空间。

```bash
$ git count-objects -v
count: 11
size: 4904
in-pack: 15
packs: 1
size-pack: 8
prune-packable: 0
garbage: 0
size-garbage: 0
```

打包的仓库大小下降到了 8K，比 5MB 好很多。 可以从 size 的值看出，这个大文件还在你的松散对象中，并没有消失；但是它不会在推送或接下来的克隆中出现，这才是最重要的。 如果真的想要删除它，可以通过有`--expire`选项的`git prune`命令来完全地移除那个对象：

```bash
$ git prune --expire now
$ git count-objects -v
count: 0
size: 0
in-pack: 15
packs: 1
size-pack: 8
prune-packable: 0
garbage: 0
size-garbage: 0
```

## 8.环境变量

### 8.1 全局行为

像通常的程序一样，Git 的常规行为依赖于环境变量。

**GIT_EXEC_PATH**决定 Git 到哪找它的子程序 （像`git-commit`，`git-diff`等等）。 你可以用`git --exec -path`来查看当前设置。

通常不会考虑修改**HOME**这个变量（太多其它东西都依赖它），这是 Git 查找全局配置文件的地方。 如果你想要一个包括全局配置的真正的便携版 Git， 你可以在便携版 Git 的 shell 配置中覆盖 HOME 设置。

**PREFIX**也类似，除了用于系统级别的配置。 Git 在 $PREFIX/etc/gitconfig 查找此文件。

如果设置了**GIT_CONFIG_NOSYSTEM**，就禁用系统级别的配置文件。 这在系统配置影响了你的命令，而你又无权限修改的时候很有用。

**GIT_PAGER**控制在命令行上显示多页输出的程序。 如果这个没有设置，就会用 PAGER 。

**GIT_EDITOR**当用户需要编辑一些文本（比如提交信息）时， Git 会启动这个编辑器。 如果没设置，就会用 EDITOR。

### 8.2 版本库位置

Git 用了几个变量来确定它如何与当前版本库交互。

**GIT_DIR**是 .git 目录的位置。 如果这个没有设置， Git 会按照目录树逐层向上查找 .git 目录，直到到达 ~或 /。

**GIT_CEILING_DIRECTORIES**控制查找 .git 目录的行为。 如果你访问加载很慢的目录（如那些磁带机上的或通过网络连接访问的），你可能会想让 Git 早点停止尝试，尤其是 shell 构建时调用了 Git 。

**GIT_WORK_TREE**是非空版本库的工作目录的根路径。 如果指定了 --git-dir 或 GIT_DIR 但未指定 --work -tree、GIT_WORK_TREE 或 core.worktree，那么当前工作目录就会视作工作树的顶级目录。

**GIT_INDEX_FILE**是索引文件的路径（只有非空版本库有）。

**GIT_OBJECT_DIRECTORY**用来指定 .git/objects 目录的位置。

**GIT_ALTERNATE_OBJECT_DIRECTORIES**一个冒号分割的列表（格式类似 /dir/one:/dir/two:…）用来告诉 Git 到哪里去找不在 GIT_OBJECT_DIRECTORY 目录中的对象。 如果你有很多项目有相同内容的大文件，这个可以用来避免存储过多备份。

### 8.3 路径规则

所谓 “pathspec” 是指你在 Git 中如何指定路径，包括通配符的使用。 它们会在 .gitignore 文件中用到，命令行里也会用到（git add *.c）。

**GIT_GLOB_PATHSPECS**和**GIT_NOGLOB_PATHSPECS**控制通配符在路径规则中的默认行为。 如果 GIT_GLOB_PATHSPECS 设置为 1，通配符表现为通配符（这是默认设置）；如果 GIT_NOGLOB_PATHSPECS 设置为 1，通配符仅匹配字面。意思是 *.c 只会匹配 文件名是 “*.c” 的文件，而不是以 .c 结尾的文件。 你可以在各个路径规范中用 :(glob) 或 :(literal) 开头来覆盖这个配置，如 :(glob)*.c 。

**GIT_LITERAL_PATHSPECS**禁用上面的两种行为；通配符将不能用，前缀覆盖也不能用。

**GIT_ICASE_PATHSPECS**让所有的路径规范忽略大小写。

### 8.4 提交

Git 提交对象的创建通常最后是由 git-commit-tree 来完成，git-commit-tree 用这些环境变量作主要的信息源。 仅当这些值不存在才回退到预置的值。

**GIT_AUTHOR_NAME**是 “author” 字段的可读名字。

**GIT_AUTHOR_EMAIL**是 “author” 字段的邮件。

**GIT_AUTHOR_DATE**是 “author” 字段的时间戳。

**GIT_COMMITTER_NAME**是 “committer” 字段的可读名字。

**GIT_COMMITTER_EMAIL**是 “committer” 字段的邮件。

**GIT_COMMITTER_DATE**是 “committer” 字段的时间戳。

如果 user.email 没有配置， 就会用到 EMAIL 指定的邮件地址。 如果 这个 也没有设置， Git 继续回退使用系统用户和主机名。

### 8.5 网络

Git 使用 curl 库通过 HTTP 来完成网络操作， 所以**GIT_CURL_VERBOSE**告诉 Git 显示所有由那个库产生的消息。 这跟在命令行执行`curl -v`差不多。

**GIT_SSL_NO_VERIFY**告诉 Git 不用验证 SSL 证书。 这在有些时候是需要的， 例如你用一个自己签名的证书通过 HTTPS 来提供 Git 服务， 或者你正在搭建 Git 服务器，还没有安装完全的证书。

如果 Git 操作在网速低于**GIT_HTTP_LOW_SPEED_LIMIT**字节／秒，并且持续 GIT_HTTP_LOW_SPEED_TIME秒以上的时间，Git 会终止那个操作。 这些值会覆盖 http.lowSpeedLimit 和 http.lowSpeedTime 配置的值。

**GIT_HTTP_USER_AGENT**设置 Git 在通过 HTTP 通讯时用到的 user-agent。 默认值类似于 git/2.0.0 。

### 8.6 比较和合并

**GIT_DIFF_OPTS**这个有点起错名字了。 有效值仅支持 -u<n> 或 --unified=<n>，用来控制在 git diff 命令中显示的内容行数。

**GIT_EXTERNAL_DIFF**用来覆盖 diff.external 配置的值。 如果设置了这个值， 当执行 git diff 时，Git会调用该程序。

**GIT_DIFF_PATH_COUNTER**和**GIT_DIFF_PATH_TOTAL**对于**GIT_EXTERNAL_DIFF**或 diff.external 指定的程序有用。 前者表示在一系列文件中哪个是被比较的（从 1 开始），后者表示每批文件的总数。

**GIT_MERGE_VERBOSITY**控制递归合并策略的输出。 允许的值有下面这些：

- 0 什么都不输出，除了可能会有一个错误信息
- 1 只显示冲突
- 2 还显示文件改变
- 3 显示因为没有改变被跳过的文件
- 4 显示处理的所有路径
- 5 显示详细的调试信息

默认值是 2。

### 8.7 调试

Git 内置了相当完整的跟踪信息，这些变量的可用值如下：

- “true”“1” 或 “2”——跟踪类别写到标准错误输出
- 以 / 开头的绝对路径——跟踪输出会被写到那个文件

**GIT_TRACE**控制常规跟踪，它并不适用于特殊情况。 它跟踪的范围包括别名的展开和其他子程序的委托。

```bash
$ GIT_TRACE=true git lga
20:12:49.877982 git.c:554               trace: exec: 'git-lga'
20:12:49.878369 run-command.c:341       trace: run_command: 'git-lga'
20:12:49.879529 git.c:282               trace: alias expansion: lga =>
'log' '--graph' '--pretty=oneline' '--abbrev-commit' '--decorate' '--all'
20:12:49.879885 git.c:349               trace: built-in: git 'log' '--
graph' '--pretty=oneline' '--abbrev-commit' '--decorate' '--all'
20:12:49.899217 run-command.c:341       trace: run_command: 'less'
20:12:49.899675 run-command.c:192       trace: exec: 'less'
```

**GIT_TRACE_PACK_ACCESS**控制访问打包文件的跟踪信息。 第一个字段是被访问的打包文件，第二个是文件的偏移量：

```bash
$ GIT_TRACE_PACK_ACCESS=true git status
20:10:12.081397 sha1_file.c:2088        .git/objects/pack/pack-
c3fa...291e.pack 12
20:10:12.081886 sha1_file.c:2088        .git/objects/pack/pack-
c3fa...291e.pack 34662
20:10:12.082115 sha1_file.c:2088        .git/objects/pack/pack-
c3fa...291e.pack 35175
# […]
20:10:12.087398 sha1_file.c:2088        .git/objects/pack/pack-
e80e...e3d2.pack 56914983
20:10:12.087419 sha1_file.c:2088        .git/objects/pack/pack-
e80e...e3d2.pack 14303666
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

**GIT_TRACE_PACKET**打开网络操作包级别的跟踪信息。

```bash
$ GIT_TRACE_PACKET=true git ls-remote origin
20:15:14.867043 pkt-line.c:46           packet:          git< #
service=git-upload-pack
20:15:14.867071 pkt-line.c:46           packet:          git< 0000
20:15:14.867079 pkt-line.c:46           packet:          git<
97b8860c071898d9e162678ea1035a8ced2f8b1f HEAD\0multi_ack thin-pack side-
band side-band-64k ofs-delta shallow no-progress include-tag
multi_ack_detailed no-done symref=HEAD:refs/heads/master agent=git/2.0.4
20:15:14.867088 pkt-line.c:46           packet:          git<
0f20ae29889d61f2e93ae00fd34f1cdb53285702 refs/heads/ab/add-interactive-
show-diff-func-name
20:15:14.867094 pkt-line.c:46           packet:          git<
36dc827bc9d17f80ed4f326de21247a5d1341fbc refs/heads/ah/doc-gitk-config
# […]
```

**GIT_TRACE_PERFORMANCE**控制性能数据的日志打印。 输出显示了每个 git 命令调用花费的时间。

```bash
$ GIT_TRACE_PERFORMANCE=true git gc
20:18:19.499676 trace.c:414             performance: 0.374835000 s: git
command: 'git' 'pack-refs' '--all' '--prune'
20:18:19.845585 trace.c:414             performance: 0.343020000 s: git
command: 'git' 'reflog' 'expire' '--all'
Counting objects: 170994, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (43413/43413), done.
Writing objects: 100% (170994/170994), done.
Total 170994 (delta 126176), reused 170524 (delta 125706)
20:18:23.567927 trace.c:414             performance: 3.715349000 s: git
command: 'git' 'pack-objects' '--keep-true-parents' '--honor-pack-keep'
'--non-empty' '--all' '--reflog' '--unpack-unreachable=2.weeks.ago' '--
local' '--delta-base-offset' '.git/objects/pack/.tmp-49190-pack'
20:18:23.584728 trace.c:414             performance: 0.000910000 s: git
command: 'git' 'prune-packed'
20:18:23.605218 trace.c:414             performance: 0.017972000 s: git
command: 'git' 'update-server-info'
20:18:23.606342 trace.c:414             performance: 3.756312000 s: git
command: 'git' 'repack' '-d' '-l' '-A' '--unpack-unreachable=2.weeks.ago'
Checking connectivity: 170994, done.
20:18:25.225424 trace.c:414             performance: 1.616423000 s: git
command: 'git' 'prune' '--expire' '2.weeks.ago'
20:18:25.232403 trace.c:414             performance: 0.001051000 s: git
command: 'git' 'rerere' 'gc'
20:18:25.233159 trace.c:414             performance: 6.112217000 s: git
command: 'git' 'gc'
```

**GIT_TRACE_SETUP**显示 Git 发现的关于版本库和交互环境的信息。

```bash
$ GIT_TRACE_SETUP=true git status
20:19:47.086765 trace.c:315             setup: git_dir: .git
20:19:47.087184 trace.c:316             setup: worktree:
/Users/ben/src/git
20:19:47.087191 trace.c:317             setup: cwd: /Users/ben/src/git
20:19:47.087194 trace.c:318             setup: prefix: (null)
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

### 8.8 其它

如果指定了**GIT_SSH**， Git 连接 SSH 主机时会用指定的程序代替 ssh 。 它会被用`$GIT_SSH [username@]host [-p <port>] <command>`的命令方式调用。 这不是配置定制 ssh 调用方式的最简单的方法; 它不支持额外的命令行参数， 所以你必须写一个封装脚本然后让 GIT_SSH 指向它。 可能用 ~/.ssh/config 会更简单。

**GIT_ASKPASS**覆盖了 core.askpass 配置。 这是 Git 需要向用户请求验证时用到的程序，它接受一个文本提示作为命令行参数，并在 stdout 中返回应答。 （查看 凭证存储 访问更多相关内容）

**GIT_NAMESPACE**控制有命令空间的引用的访问，与`--namespace`标志是相同的。 这主要在服务器端有用，如果你想在一个版本库中存储单个版本库的多个 fork，只要保持引用是隔离的就可以。

**GIT_FLUSH**强制 Git 在向标准输出增量写入时使用没有缓存的 I/O。 设置为 1 让 Git 刷新更多， 设置为 0 则使所有的输出被缓存。 默认值（若此变量未设置）是根据活动和输出模式的不同选择合适的缓存方案。

**GIT_REFLOG_ACTION**让你可以指定描述性的文字写到 reflog 中。 这有个例子：

```bash
$ GIT_REFLOG_ACTION="my action" git commit --allow-empty -m 'my message'
[master 9e3d55a] my message
$ git reflog -1
9e3d55a HEAD@{0}: my action: my message
```

