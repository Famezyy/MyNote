# 第8章_自定义Git

## 1.配置Git

可以用`git config`配置 Git。首先要做的事情就是设置你的名字和邮件地址：

```bash
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```

首先，快速回忆下：Git 使用一系列配置文件来保存你自定义的行为。它首先会查找系统级的 /etc/gitconfig 文件，该文件含有系统里每位用户及他们所拥有的仓库的配置值。如果你传递`--system`选项给`git config`，它就会读写该文件。

接下来 Git 会查找每个用户的 ~/.gitconfig 文件（或者 ~/.config/git/config 文件）。你可以传递`--global`选项让 Git 读写该文件。

最后 Git 会查找你正在操作的仓库所对应的 Git 目录下的配置文件（.git/config）。这个文件中的值只对该仓库有效，它对应于向`git config`传递`--local`选项。

以上三个层次中每层的配置（系统、全局、本地）都会覆盖掉上一层次的配置，所以 .git/config 中的值会覆盖掉 /etc/gitconfig 中所对应的值。

### 1.1 客户端基本配置

Git 能够识别的配置项分为两大类：客户端和服务器端。其中大部分属于客户端配置。我们只讲述最平常和最有用的选项。如果想得到你当前版本的 Git 支持的选项列表，请运行：

```bash
$ man git-config
```

这个命令列出了所有可用的选项，以及与之相关的介绍。你也可以在 http://git-scm.com/docs/git-config 找到同样的内容。

**core.editor**

默认情况下，Git 会调用你通过环境变量`$VISUAL`或`$EDITOR`设置的文本编辑器，如果没有设置，默认则会调用 vi 来创建和编辑你的提交以及标签信息。你可以使用`core.editor`选项来修改默认的编辑器：

```bash
$ git config --global core.editor emacs
```

现在，无论你定义了什么终端编辑器，Git 都会调用 Emacs 编辑信息。

**commit.template**

如果把此项指定为你的系统上某个文件的路径，当你提交的时候，Git 会使用该文件的内容作为提交的默认初始化信息。创建的自定义提交模版中的值可以用来提示自己或他人适当的提交格式和风格。

例如：考虑 ~/.gitmessage.txt 模板文件：

```bash
Subject line (try to keep under 50 characters)

Multi-line description of commit,
feel free to be detailed.

[Ticket: X]
```

注意此提交模版是如何提示提交者保持主题的简短（为了精简`git log --oneline`的输出），如何在后面添加进一步的详情，如何引用问题和 bug 跟踪系统的工单号（Ticket），如果有的话。

要想让 Git 把它作为运行`git commit`时显示在你的编辑器中的默认信息，如下设置`commit.template`：

```bash
$ git config --global commit.template ~/.gitmessage.txt
$ git commit
```

当你提交时，编辑器中就会显示如下的提交信息占位符：

```bash
Subject line (try to keep under 50 characters)

Multi-line description of commit,
feel free to be detailed.

[Ticket: X]
# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
# modified:   lib/test.rb
#
~
~
".git/COMMIT_EDITMSG" 14L, 297C
```

如果你的团队对提交信息有格式要求，可以在系统上创建一个文件，并配置 Git 把它作为默认的模板，这样可以更加容易地使提交信息遵循格式。

**core.pager**

该配置项指定 Git 运行诸如 log 和 diff 等命令所使用的分页器。你可以把它设置成用 more 或者任何你喜欢的分页器（默认用的是 less），当然也可以设置成空字符串，关闭该选项：

```bash
$ git config --global core.pager ''
```

这样不管命令的输出量多少，Git 都会在一页显示所有内容。

**user.signingkey**

如果你要创建经签署的含附注的标签，那么把你的 GPG 签署密钥设置为配置项会更好。如下设置你的密钥 ID：

```bash
$ git config --global user.signingkey <gpg-key-id>
```

现在，你每次运行 git tag 命令时，即可直接签署标签，而无需定义密钥：

```bash
$ git tag -s <tag-name>
```

**core.excludesfile**

你可以在你的项目的 .gitignore 文件里面规定无需纳入 Git 管理的文件的模板，这样它们既不会出现在未跟踪列表，也不会在你运行`git add`后被暂存。

不过有些时候，你想要在你所有的版本库中忽略掉某一类文件。如果你的操作系统是 macOS，很可能就是指 .DS_Store。如果你把 Emacs 或 Vim 作为首选的编辑器，你肯定知道以 ~ 结尾的文件名。

这个配置允许你设置类似于全局生效的 .gitignore 文件。如果你按照下面的内容创建一个 ~/.gitignore_global 文件：

```bash
*~
.*.swp
.DS_Store
```

然后运行`git config --global core.excludesfile ~/.gitignore_global`，Git 将把那些文件永远地拒之门外。

**help.autocorrect**

假如你打错了一条命令，会显示：

```bash
$ git chekcout master
git：'chekcout' 不是一个 git 命令。参见 'git --help'。
您指的是这个么？
  checkout
```

Git 会尝试猜测你的意图，但是它不会越俎代庖。如果你把`help.autocorrect`设置成 1，那么只要有一个命令被模糊匹配到了，Git 会自动运行该命令。

```bash
$ git chekcout master
警告：您运行一个不存在的 Git 命令 'chekcout'。继续执行假定您要要运行的是 'checkout'
在 0.1 秒钟后自动运行...
```

注意提示信息中的“0.1 秒”。help.autocorrect 接受一个代表十分之一秒的整数。所以如果你把它设置为50，Git 将在自动执行命令前给你 5 秒的时间改变主意。

### 1.2 Git中的着色

Git 充分支持对终端内容着色，对你凭肉眼简单、快速分析命令输出有很大帮助。

**color.ui**

Git 会自动着色大部分输出内容，但如果你不喜欢花花绿绿，也可以关掉。要想关掉 Git 的终端颜色输出，试一下这个：

```bash
$ git config --global color.ui false
```

这个设置的默认值是 auto，它会着色直接输出到终端的内容；而当内容被重定向到一个管道或文件时，则忽略着色功能。

你也可以设置成 always，来忽略掉管道和终端的不同，即在任何情况下着色输出。你很少会这么设置，在大多数场合下，如果你想在被重定向的输出中插入颜色码，可以传递`--color`标志给 Git 命令来强制它这么做。默认设置就已经能满足大多数情况下的需求了。

**color.***

要想具体到哪些命令输出需要被着色以及怎样着色，你需要用到和具体命令有关的颜色配置选项。它们都能被置为 true、false 或 always：

```bash
color.branch
color.diff
color.interactive
color.status
```

另外，以上每个配置项都有子选项，它们可以被用来覆盖其父设置，以达到为输出的各个部分着色的目的。例如，为了让 diff 的输出信息以蓝色前景、黑色背景和粗体显示，你可以运行

```bash
$ git config --global color.diff.meta "blue black bold"
```

你能设置的颜色有：normal、black、red、green、yellow、blue、magenta、cyan 或 white。正如以上例子设置的粗体属性，想要设置字体属性的话，可以选择包括：bold、dim、ul（下划线）、blink、reverse（交换前景色和背景色）。

### 1.3 外部的合并与比较工具

 虽然 Git 自己内置了一个 diff 实现，而且到目前为止我们一直在使用它，但你能够用一个外部的工具替代它。除此以外，你还能设置一个图形化的工具来合并和解决冲突，从而不必自己手动解决。这里我们以一个不错且免费的工具 —— Perforce 图形化合并工具（P4Merge）—— 来展示如何用一个外部的工具来合并和解决冲突。

P4Merge 可以在所有主流平台上运行，所以安装上应该没有什么困难。在这个例子中，我们使用的路径名可以直接应用在 macOS 和 Linux 上； 在 Windows 上，/usr/local/bin 需要被改为你的环境中可执行文件所在的目录路径。

首先，[下载 P4Merge](https://www.perforce.com/product/components/perforce-visual-merge-and-diff-tools)。接下来，你要编写一个全局包装脚本来运行你的命令。我们会使用 Mac 上的路径来指定该脚本的位置，在其他系统上，它将是 p4merge 二进制文件所在的目录。创建一个名为 extMerge 的脚本包装 merge 命令，让它把参数转发给 p4merge 二进制文件：

```bash
$ cat /usr/local/bin/extMerge
#!/bin/sh
/Applications/p4merge.app/Contents/MacOS/p4merge $*
```

包装 diff 命令的脚本首先确保传递了七个参数过来，随后把其中两个转发给包装了 merge 的脚本。默认情况下，Git 传递以下参数给 diff：

```bash
path old-file old-hex old-mode new-file new-hex new-mode
```

由于你仅仅需要`old-file`和`new-file`参数，由包装 diff 的脚本来转发它们吧。

```basj
$ cat /usr/local/bin/extDiff
#!/bin/sh
[ $# -eq 7 ] && /usr/local/bin/extMerge "$2" "$5"
```

你也需要确保这些脚本具有可执行权限：

```bash
$ sudo chmod +x /usr/local/bin/extMerge
$ sudo chmod +x /usr/local/bin/extDiff
```

现在你可以修改配置文件来使用你自定义的合并和比较工具了。这将涉及许多自定义设置：merge.tool 通知 Git 该使用哪个合并工具，`mergetool.<tool>.cmd`规定命令运行的方式，`mergetool.<tool>.trustExitCode`会通知 Git 程序的返回值是否表示合并操作成功，`diff.external`通知 Git 该用什么命令做比较。因此，你可以运行以下四条配置命令：

```bash
$ git config --global merge.tool extMerge
$ git config --global mergetool.extMerge.cmd \
  'extMerge "$BASE" "$LOCAL" "$REMOTE" "$MERGED"'
$ git config --global mergetool.extMerge.trustExitCode false
$ git config --global diff.external extDiff
```

或编辑你的 ~/.gitconfig 文件，添加以下各行：

```bash
[merge]
  tool = extMerge
[mergetool "extMerge"]
  cmd = extMerge "$BASE" "$LOCAL" "$REMOTE" "$MERGED"
  trustExitCode = false
[diff]
  external = extDiff
```

待一切设置妥当后，如果你像这样运行 diff 命令：

```basj
$ git diff 32d1776b1^ 32d1776b1
```

Git 将启动 P4Merge，而不是在命令行输出比较的结果，就像这样：

<img src="img/第8章_自定义GIT/image-20220707230332644.png" alt="image-20220707230332644" style="zoom:80%;" />

如果你尝试合并两个分支，随后遇到了合并冲突，运行 git mergetool，Git 会调用 P4Merge 让你通过图形界面来解决冲突。

设置包装脚本的好处在于大大降低了改变 diff 和 merge 工具的工作量。举个例子，想把 extDiff 和 extMerge 的工具改成 KDiff3，你要做的仅仅是编辑 extMerge 脚本文件：

```bash
$ cat /usr/local/bin/extMerge
#!/bin/sh
/Applications/kdiff3.app/Contents/MacOS/kdiff3 $*
```

现在，Git 将使用 KDiff3 作为查看比较和解决合并冲突的工具。

Git 预设了许多其他的合并和解决冲突的工具，无需特别的设置你就能用上它们。要想看到它支持的工具列表，试一下这个：

```bash
$ git mergetool --tool-help
'git mergetool --tool=<tool>' may be set to one of the following:
        emerge
        gvimdiff
        gvimdiff2
        opendiff
        p4merge
        vimdiff
        vimdiff2

The following tools are valid, but not currently available:
        araxis
        bc3
        codecompare
        deltawalker
        diffmerge
        diffuse
        ecmerge
        kdiff3
        meld
        tkdiff
        tortoisemerge
        xxdiff

Some of the tools listed above only work in a windowed
environment. If run in a terminal-only session, they will fail.
```

如果你不想用到 KDiff3 的所有功能，只是想用它来合并，那么 kdiff3 正符合你的要求，运行：

```bash
$ git config --global merge.tool kdiff3
```

如果运行了以上命令，而没有设置 extMerge 和 extDiff 文件，Git 会用 KDiff3 做合并，让内置的 diff 来做比较。

### 1.4 格式化与多余的空白字符

格式化与多余的空白字符是许多开发人员在协作时，特别是在跨平台情况下，不时会遇到的令人头疼的琐碎的问题。由于编辑器的不同或者文件行尾的换行符在 Windows 下被替换了，一些细微的空格变化会不经意地混入提交的补丁或其它协作成果中。不用怕，Git 提供了一些配置项来帮助你解决这些问题。

**core.autocrlf**

假如你正在 Windows 上写程序，而你的同伴用的是其他系统（或相反），你可能会遇到 CRLF 问题。这是因为 Windows 使用回车（CR）和换行（LF）两个字符来结束一行，而 macOS 和 Linux 只使用换行（LF）一个字符。虽然这是小问题，但它会极大地扰乱跨平台协作。许多 Windows 上的编辑器会悄悄把行尾的换行字符转换成回车和换行，或在用户按下 Enter 键时，插入回车和换行两个字符。

Git 可以在你提交时自动地把回车和换行转换成换行，而在检出代码时把换行转换成回车和换行。你可以用`core.autocrlf`来打开此项功能。如果是在 Windows 系统上，把它设置成 true，这样在检出代码时，换行会被转换成回车和换行：

```bash
$ git config --global core.autocrlf true
```

如果使用以换行作为行结束符的 Linux 或 macOS，你不需要 Git 在检出文件时进行自动的转换； 然而当一个以回车加换行作为行结束符的文件不小心被引入时，你肯定想让 Git 修正。你可以把`core.autocrlf`设置成 input 来告诉 Git 在提交时把回车和换行转换成换行，检出时不转换：

```bash
$ git config --global core.autocrlf input
```

这样在 Windows 上的检出文件中会保留回车和换行，而在 macOS 和 Linux 上，以及版本库中会保留换行。

如果你是 Windows 程序员，且正在开发仅运行在 Windows 上的项目，可以设置 false 取消此功能，把回车保留在版本库中：

```bash
$ git config --global core.autocrlf false
```

Git 预先设置了一些选项来探测和修正多余空白字符问题。它提供了六种处理多余空白字符的主要选项 —— 其中三个默认开启，另外三个默认关闭，不过你可以自由地设置它们。

默认被打开的三个选项是：`blank-at-eol`，查找行尾的空格；`blank-at-eof`，盯住文件底部的空行；`space-before-tab`，警惕行头 tab 前面的空格。

默认被关闭的三个选项是：`indent-with-non-tab`，揪出以空格而非 tab 开头的行（你可以用 tabwidth 选项控制它）；`tab-in-indent`，监视在行头表示缩进的 tab；`cr-at-eol`，告诉 Git 忽略行尾的回车。

通过设置`core.whitespace`，你可以让 Git 按照你的意图来打开或关闭以逗号分割的选项。要想关闭某个选项，你可以在输入设置选项时不指定它或在它前面加个`-`。例如，如果你想要打开除`space-before-tab`之外的所有选项，那么可以这样（`trailing-space`涵盖了`blank-at-eol`和`blank-at-eof`）：

```bash
$ git config --global core.whitespace trailing-space,-space-before-tab,indent-with-non-tab,tab-in-indent,cr-at-eol
```

你也可以只指定自定义的部分：

```bash
$ git config --global core.whitespace -space-before-tab,indent-with-non-tab,tab-in-indent,cr-at-eol
```

当你运行`git diff`命令并尝试给输出着色时，Git 将探测到这些问题，因此你在提交前就能修复它们。用`git apply`打补丁时你也会从中受益。如果正准备应用的补丁存有特定的空白问题，你可以让 Git 在应用补丁时发出警告：

```bash
$ git apply --whitespace=warn <patch>
```

或者让 Git 在打上补丁前自动修正此问题：

```basj
$ git apply --whitespace=fix <patch>
```

这些选项也能运用于`git rebase`。如果提交了有空白问题的文件，但还没推送到上游，你可以运行`git rebase --whitespace=fix`来让 Git 在重写补丁时自动修正它们。

### 1.5 服务器端配置

Git 服务器端的配置项相对来说并不多，但仍有一些饶有生趣的选项值得你一看。

**receive.fsckObjects**

Git 能够确认每个对象的有效性以及 SHA-1 检验和是否保持一致。但 Git 不会在每次推送时都这么做。这个操作很耗时间，很有可能会拖慢提交的过程，特别是当库或推送的文件很大的情况下。如果想在每次推送时都要求 Git 检查一致性，设置`receive.fsckObjects`为 true 来强迫它这么做：

```bash
$ git config --system receive.fsckObjects true
```

现在 Git 会在每次推送生效前检查库的完整性，确保没有被有问题的客户端引入破坏性数据。

**receive.denyNonFastForwards**

如果你变基已经被推送的提交，继而再推送，又或者推送一个提交到远程分支，而这个远程分支当前指向的提交不在该提交的历史中，这样的推送会被拒绝。这通常是个很好的策略，但有时在变基的过程中，你确信自己需要更新远程分支，可以在`push`命令后加`-f`标志来强制更新（force-update）。

要禁用这样的强制更新推送（force-pushes），可以设置`receive.denyNonFastForwards`：

```bash
$ git config --system receive.denyNonFastForwards true
```

稍后我们会提到，用服务器端的接收钩子也能达到同样的目的。那种方法可以做到更细致的控制，例如禁止某一类用户做非快进（non-fast-forwards）推送。

**receive.denyDeletes**

有一些方法可以绕过`denyNonFastForwards`策略。其中一种是先删除某个分支，再连同新的引用一起推送回该分支。把`receive.denyDeletes`设置为 true 可以把这个漏洞补上：

```bash
$ git config --system receive.denyDeletes true
```

这样会禁止通过推送删除分支和标签 — 没有用户可以这么做。要删除远程分支，必须从服务器手动删除引用文件。通过用户访问控制列表（ACL）也能够在用户级的粒度上实现同样的功能，你将在 使用强制策略的一个例子 一节学到具体的做法。

## 2.Git属性

你也可以针对特定的路径配置某些设置项，这样 Git 就只对特定的子目录或子文件集运用它们。这些基于路径的设置项被称为 Git 属性，可以在你的目录下的 .gitattributes 文件内进行设置（通常是你的项目的根目录）。如果不想让这些属性文件与其它文件一同提交，你也可以在 .git/info/attributes 文件中进行设置。

通过使用属性，你可以对项目中的文件或目录单独定义不同的合并策略，让 Git 知道怎样比较非文本文件，或者让 Git 在提交或检出前过滤内容。在本节，你将学习到一些能在自己的项目中用到的属性，并看到几个实际的例子。

### 2.1 二进制文件

你可以用 Git 属性让 Git 知道哪些是二进制文件（以防它没有识别出来），并指示其如何处理这些文件。例如，一些文本文件是由机器产生的，没有办法进行比较，但是一些二进制文件可以比较。你将了解到怎样让 Git 区分这些文件。

#### 1.识别二进制文件

有些文件表面上是文本文件，实质上应被作为二进制文件处理。例如，macOS 平台上的 Xcode 项目会包含一个以 .pbxproj 结尾的文件，它通常是一个记录项目构建配置等信息的 JSON（纯文本 Javascript 数据类型）数据集，由 IDE 写入磁盘。虽然技术上看它是由 UTF-8 编码的文本文件，但你并不会希望将它当作文本文件来处理，因为它其实是一个轻量级数据库——如果有两个人修改了它，你通常无法合并内容，diff 的输出也帮不上什么忙。它本应被机器处理。因此，你想把它当成二进制文件。

要让 Git 把所有 pbxproj 文件当成二进制文件，在 .gitattributes 文件中如下设置：

```bash
*.pbxproj binary
```

现在，Git 不会尝试转换或修正回车换行（CRLF）问题，当你在项目中运行`git show`或`git diff`时，Git 也不会比较或打印该文件的变化。

#### 2.比较二进制文件

你也可以使用 Git 属性来有效地比较两个二进制文件。秘诀在于，告诉 Git 怎么把你的二进制文件转化为文本格式，从而能够使用普通的 diff 方式进行对比。

首先，让我们尝试用这个技术解决世人最头疼的问题之一：对 Microsoft Word 文档进行版本控制。大家都知道，Microsoft Word 几乎是世上最难缠的编辑器，尽管如此，大家还是在用它。如果想对 Word 文档进行版本控制，你可以把文件加入到 Git 库中，每次修改后提交即可。但这样做有什么实际意义呢？毕竟运行`git diff`命令后，你只能得到如下的结果：

```bash
$ git diff
diff --git a/chapter1.docx b/chapter1.docx
index 88839c4..4afcb7c 100644
Binary files a/chapter1.docx and b/chapter1.docx differ
```

除了检出之后睁大眼睛逐行扫描，就真的没有办法直接比较两个不同版本的 Word 文档吗？Git 属性能很好地解决此问题。把下面这行文本加到你的 .gitattributes 文件中：

```bash
*.docx diff=word
```

这告诉 Git 当你尝试查看包含变更的比较结果时，所有匹配 .docx 模式的文件都应该使用“word”过滤器。“word”过滤器是什么？我们现在就来设置它。我们会对 Git 进行配置，令其能够借助 docx2txt 程序将 Word 文档转为可读文本文件，这样不同的文件间就能够正确比较了。

首先，你需要安装**docx2txt**；它可以从 https://sourceforge.net/projects/docx2txt 下载。按照 INSTALL 文件的说明，把它放到你的可执行路径下。接下来，你还需要写一个脚本把输出结果包装成 Git 支持的格式。在你的可执行路径下创建一个叫 docx2txt 文件，添加这些内容：

```bash
#!/bin/bash
docx2txt.pl "$1" -
```

别忘了用 chmod a+x 给这个文件加上可执行权限。最后，你需要配置 Git 来使用这个脚本：

```bash
$ git config diff.word.textconv docx2txt
```

现在如果在两个快照之间进行比较，Git 就会对那些以 .docx 结尾的文件应用“word”过滤器，即 docx2txt。这样你的 Word 文件就能被高效地转换成文本文件并进行比较了。

你还能用这个方法比较图像文件。其中一个办法是，在比较时对图像文件运用一个过滤器，提炼出 EXIF 信息——这是在大部分图像格式中都有记录的一种元数据。如果你下载并安装了**exiftool**程序，可以利用它将图像转换为关于元数据的文本信息，这样比较时至少能以文本的形式显示发生过的变动：将以下内容放到你的 .gitattributes 文件中：

```bash
*.png diff=exif
```

配置 Git 以使用此工具：

```bash
$ git config diff.exif.textconv exiftool
```

如果在项目中替换了一个图像文件，运行`git diff`命令的结果如下：

```bash
diff --git a/image.png b/image.png
index 88839c4..4afcb7c 100644
--- a/image.png
+++ b/image.png
@@ -1,12 +1,12 @@
 ExifTool Version Number         : 7.74
-File Size                       : 70 kB
-File Modification Date/Time     : 2009:04:21 07:02:45-07:00
+File Size                       : 94 kB
+File Modification Date/Time     : 2009:04:21 07:02:43-07:00
 File Type                       : PNG
 MIME Type                       : image/png
-Image Width                     : 1058
-Image Height                    : 889
+Image Width                     : 1056
+Image Height                    : 827
 Bit Depth                       : 8
 Color Type                      : RGB with Alpha
```

### 2.2 关键字展开

SVN 或 CVS 风格的关键字展开（keyword expansion）功能经常会被习惯于上述系统的开发者使用到。在 Git中，这项功能有一个主要问题，就是你无法利用它往文件中加入其关联提交的相关信息，因为 Git 总是先对文件做校验和运算（译者注：Git 中提交对象的校验依赖于文件的校验和，而 Git 属性针对特定文件或路径，因此基于 Git 属性的关键字展开无法仅根据文件反推出对应的提交）。不过，我们可以在检出某个文件后对其注入文本，并在再次提交前删除这些文本。Git 属性提供了两种方法来达到这一目的。

一种方法是，你可以把文件所对应数据对象的 SHA-1 校验和自动注入到文件中的`$Id$`字段。如果在一个或多个文件上设置了该属性，下次当你检出相关分支的时候，Git 会用相应数据对象的 SHA-1 值替换上述字段。注意，这不是提交对象的 SHA-1 校验和，而是数据对象本身的校验和。将以下内容放到你的 .gitattributes文件中：

```bash
*.txt ident
```

在一个测试文件中添加一个`$Id$`引用：

```bash
$ echo '$Id$' > test.txt
```

当你下次检出文件时，Git 将注入数据对象的 SHA-1 校验和：

```bash
$ rm test.txt
$ git checkout -- test.txt
$ cat test.txt
$Id: 42812b7653c7b88933f8a9d6cad0ca16714b9bb3 $
```

然而，这个结果的用途比较有限。如果用过 CVS 或 Subversion 的关键字替换功能，我们会想加上一个时间戳信息——光有 SHA-1 校验和用途不大，它仅仅是个随机字符串，你无法凭字面值来区分不同 SHA-1 时间上的先后。

因此 Git 属性提供了另一种方法：我们可以编写自己的过滤器来实现文件提交或检出时的关键字替换。一个过滤器由“clean”和“smudge”两个子过滤器组成。在 .gitattributes 文件中，你能对特定的路径设置一个过滤器，然后设置文件检出前的处理脚本（“smudge” 过滤器会在文件被检出时触发）和文件暂存前的处理脚本（“clean” 过滤器会在文件被暂存时触发）。这两个过滤器能够被用来做各种有趣的事。

<img src="img/第8章_自定义GIT/image-20220707232525438.png" alt="image-20220707232525438" style="zoom:33%;" />

<img src="img/第8章_自定义GIT/image-20220707232555335.png" alt="image-20220707232555335" style="zoom:33%;" />

在（Git 源码中）实现这个特性的原始提交信息里给出了一个简单的例子：在提交前，用 indent 程序过滤所有 C 源码。你可以在 .gitattributes 文件中对 filter 属性设置 “indent” 过滤器来过滤 *.c 文件：

```bash
*.c filter=indent
```

然后，通过以下配置，让 Git 知道“indent”过滤器在 smudge 和 clean 时分别该做什么：

```bash
$ git config --global filter.indent.clean indent
$ git config --global filter.indent.smudge cat
```

在这个例子中，当你暂存 *.c 文件时，indent 程序会先被触发；在把它们检出回硬盘时，cat 程序会先被触发。cat 在这里没什么实际作用：它仅仅把输入的数据重新输出。这样的组合可以有效地在暂存前用 indent 过滤所有的 C 源码。

另一个有趣的例子是实现 RCS 风格的`$Date$`关键字展开。要想演示这个例子，我们需要实现这样的一个小脚本：接受文件名参数，得到项目的最新提交日期，并把日期写入该文件。下面是一个实现了该功能的 Ruby 小脚本：

```bash
#! /usr/bin/env ruby
data = STDIN.read
last_date = `git log --pretty=format:"%ad" -1`
puts data.gsub('$Date$', '$Date: ' + last_date.to_s + '$')
```

这个脚本从`git log`中得到最新提交日期，将其注入所有输入文件的`$Date$`字段，并输出结果——你可以使用最顺手的语言轻松实现一个类似的脚本。把该脚本命名为 expand_date，放到你的可执行路径中。现在，你需要在 Git 中设置一个过滤器（就叫它 dater 吧），让它在检出文件时调用你的 expand_date 来注入时间戳，完成 smudge 操作。暂存文件时的 clean 操作则是用一行 Perl 表达式清除注入的内容：

```bash
$ git config filter.dater.smudge expand_date
$ git config filter.dater.clean perl -pe "s/\\\$Date[^\\\$]*\\\$/\\\$Date\\\$/"
```

这段 Perl 代码会删除`$Date$`后面注入的内容，恢复它的原貌。过滤器终于准备完成了，是时候测试一下。创建一个带有`$Date$`关键字的文件，然后给它设置一个 Git 属性，关联我们的新过滤器：

```bash
date*.txt filter=dater
```

```bash
$ echo '# $Date$' > date_test.txt
```

提交该文件，并再次检出，你会发现关键字如期被替换了：

```bash
$ git add date_test.txt .gitattributes
$ git commit -m "Testing date expansion in Git"
$ rm date_test.txt
$ git checkout date_test.txt
$ cat date_test.txt
# $Date: Tue Apr 21 07:26:52 2009 -0700$
```

自定义过滤器真的很强大。不过你需要注意的是，因为 .gitattributes 文件会随着项目一起提交，而过滤器（例如这里的 dater）不会，所以过滤器有可能会失效。当你在设计这些过滤器时，要注重容错性——它们在出错时应该能优雅地退出，从而不至于影响项目的正常运行。

### 2.3 导出版本库

Git 属性在导出项目归档（archive）时也能发挥作用。

**export-ignore**

当归档的时候，可以设置 Git 不导出某些文件和目录。如果你不想在归档中包含某个子目录或文件，但想把它们纳入项目的版本管理中，你可以在 export-ignore 属性中指定它们。

例如，假设你在 test/ 子目录下有一些测试文件，不希望它们被包含在项目导出的压缩包（tarball）中。你可以增加下面这行到 Git 属性文件中：

```bash
test/ export-ignore
```

现在，当你运行`git archive`来创建项目的压缩包时，那个目录不会被包括在归档中。

**export-subst**

在导出文件进行部署的时候，你可以将`git log`的格式化和关键字展开处理应用到标记了 export-subst 属性的部分文件。

举个例子，如果你想在项目中包含一个叫做 LAST_COMMIT 的文件，并在运行`git archive`的时候自动向它注入最新提交的元数据，可以像这样设置 .gitattributes 和 LAST_COMMIT 该文件：

```bash
LAST_COMMIT export-subst
```

```bash
$ echo 'Last commit date: $Format:%cd by %aN$' > LAST_COMMIT
$ git add LAST_COMMIT .gitattributes
$ git commit -am 'adding LAST_COMMIT file for archives'
```

运行 git archive 之后，归档文件的内容会被替换成这样：

```bash
$ git archive HEAD | tar xCf ../deployment-testing -
$ cat ../deployment-testing/LAST_COMMIT
Last commit date: Tue Apr 21 08:38:48 2009 -0700 by Scott Chacon
```

你也可以用诸如提交信息或者任意的`git notes`进行替换，并且`git log`还能做简单的字词折行：

```bash
$ echo '$Format:Last commit: %h by %aN at %cd%n%+w(76,6,9)%B$' >
LAST_COMMIT
$ git commit -am 'export-subst uses git log'\''s custom formatter
git archive 直接使用 git log 的 `pretty=format:`
处理器，并在输出中移除两侧的 `$Format:` 和 `$`
标记。
'
$ git archive @ | tar xfO - LAST_COMMIT
Last commit: 312ccc8 by Jim Hill at Fri May 8 09:14:04 2015 -0700
       export-subst uses git log's custom formatter

         git archive uses git log's `pretty=format:` processor directly,
and
         strips the surrounding `$Format:` and `$` markup from the output.
```

由此得到的归档适用于（当前的）部署工作。然而和其他的导出归档一样，它并不适用于后继的部署工作。

### 2.4 合并策略

通过 Git 属性，你还能对项目中的特定文件指定不同的合并策略。一个非常有用的选项就是，告诉 Git 当特定文件发生冲突时不要尝试合并它们，而是直接使用你这边的内容。

考虑如下场景：项目中有一个分叉的或者定制过的主题分支，你希望该分支上的更改能合并回你的主干分支，同时需要忽略其中某些文件。此时这个合并策略就能派上用场。假设你有一个数据库设置文件 database.xml，在两个分支中它是不同的，而你想合并另一个分支到你的分支上，又不想弄乱该数据库文件。你可以设置属性如下：

```bash
database.xml merge=ours
```

然后定义一个虚拟的合并策略，叫做 ours：

```bash
$ git config --global merge.ours.driver true
```

如果你合并了另一个分支，database.xml 文件不会有合并冲突，相反会显示如下信息：

```bash
$ git merge topic
Auto-merging database.xml
Merge made by recursive.
```

这里，database.xml 保持了主干分支中的原始版本。

## 3.Git钩子

和其它版本控制系统一样，Git 能在特定的重要动作发生时触发自定义脚本。有两组这样的钩子：客户端的和服务器端的。客户端钩子由诸如提交和合并这样的操作所调用，而服务器端钩子作用于诸如接收被推送的提交这样的联网操作。你可以随心所欲地运用这些钩子。

### 3.1 安装一个钩子

钩子都被存储在 Git 目录下的 hooks 子目录中。也即绝大部分项目中的 .git/hooks。当你用`git init`初始化一个新版本库时，Git 默认会在这个目录中放置一些示例脚本。这些脚本除了本身可以被调用外，它们还透露了被触发时所传入的参数。所有的示例都是 shell 脚本，其中一些还混杂了 Perl 代码，不过，任何正确命名的可执行脚本都可以正常使用 —— 你可以用 Ruby 或 Python，或任何你熟悉的语言编写它们。这些示例的名字都是以 .sample 结尾，如果你想启用它们，得先移除这个后缀。

把一个正确命名（不带扩展名）且可执行的文件放入 .git 目录下的 hooks 子目录中，即可激活该钩子脚本。这样一来，它就能被 Git 调用。接下来，我们会讲解常用的钩子脚本类型。

### 3.2 客户端钩子

客户端钩子分为很多种。下面把它们分为：**提交工作流钩子**、**电子邮件工作流钩子**和**其它钩子**。

> 需要注意的是，克隆某个版本库时，它的客户端钩子并不随同复制。如果需要靠这些脚本来强制维持某种策略，建议你在服务器端实现这一功能。

#### 1.提交工作流钩子

前四个钩子涉及提交的过程。

**pre-commit**钩子在键入提交信息前运行。它用于检查即将提交的快照，例如，检查是否有所遗漏，确保测试运行，以及核查代码。如果该钩子以非零值退出，Git 将放弃此次提交，不过你可以用`git commit --no -verify`来绕过这个环节。你可以利用该钩子，来检查代码风格是否一致（运行类似 lint 的程序）、尾随空白字符是否存在（自带的钩子就是这么做的），或新方法的文档是否适当。

**prepare-commit-msg**钩子在启动提交信息编辑器之前，默认信息被创建之后运行。它允许你编辑提交者所看到的默认信息。该钩子接收一些选项：存有当前提交信息的文件的路径、提交类型和修补提交的提交的 SHA-1 校验。它对一般的提交来说并没有什么用；然而对那些会自动产生默认信息的提交，如提交信息模板、合并提交、压缩提交和修订提交等非常实用。你可以结合提交模板来使用它，动态地插入信息。

**commit-msg**钩子接收一个参数，此参数即上文提到的，存有当前提交信息的临时文件的路径。如果该钩子脚本以非零值退出，Git 将放弃提交，因此，可以用来在提交通过前验证项目状态或提交信息。在本章的最后一节，我们将展示如何使用该钩子来核对提交信息是否遵循指定的模板。

**post-commit**钩子在整个提交过程完成后运行。它不接收任何参数，但你可以很容易地通过运行`git log -1 HEAD`来获得最后一次的提交信息。该钩子一般用于通知之类的事情。

#### 2.电子邮件工作流钩子

你可以给电子邮件工作流设置三个客户端钩子。它们都是由`git am`命令调用的，因此如果你没有在你的工作流中用到这个命令，可以跳到下一节。如果你需要通过电子邮件接收由`git format-patch`产生的补丁，这些钩子也许用得上。

第一个运行的钩子是**applypatch-msg**。它接收单个参数：包含请求合并信息的临时文件的名字。如果脚本返回非零值，Git 将放弃该补丁。你可以用该脚本来确保提交信息符合格式，或直接用脚本修正格式错误。

下一个在`git am`运行期间被调用的是**pre-applypatch**。有些难以理解的是，它正好运行于应用补丁之后，产生提交之前，所以你可以用它在提交前检查快照。你可以用这个脚本运行测试或检查工作区。如果有什么遗漏，或测试未能通过，脚本会以非零值退出，中断`git am`的运行，这样补丁就不会被提交。

**post-applypatch**运行于提交产生之后，是在`git am`运行期间最后被调用的钩子。你可以用它把结果通知给一个小组或所拉取的补丁的作者。但你没办法用它停止打补丁的过程。

#### 3.其它客户端钩子

**pre-rebase**钩子运行于变基之前，以非零值退出可以中止变基的过程。你可以使用这个钩子来禁止对已经推送的提交变基。Git 自带的**pre-rebase**钩子示例就是这么做的，不过它所做的一些假设可能与你的工作流程不匹配。

**post-rewrite**钩子被那些会替换提交记录的命令调用，比如`git commit --amend`和`git rebase`（不过不包括`git filter-branch`）。它唯一的参数是触发重写的命令名，同时从标准输入中接受一系列重写的提交记录。这个钩子的用途很大程度上跟**post-checkout**和**post-merge**差不多。

在`git checkout`成功运行后，**post-checkout**钩子会被调用。你可以根据你的项目环境用它调整你的工作目录。其中包括放入大的二进制文件、自动生成文档或进行其他类似这样的操作。

在`git merge`成功运行后，**post-merge**钩子会被调用。你可以用它恢复 Git 无法跟踪的工作区数据，比如权限数据。这个钩子也可以用来验证某些在 Git 控制之外的文件是否存在，这样你就能在工作区改变时，把这些文件复制进来。

**pre-push**钩子会在`git push`运行期间，更新了远程引用但尚未传送对象时被调用。它接受远程分支的名字和位置作为参数，同时从标准输入中读取一系列待更新的引用。你可以在推送开始之前，用它验证对引用的更新操作（一个非零的退出码将终止推送过程）。

Git 的一些日常操作在运行时，偶尔会调用`git gc --auto`进行垃圾回收。**pre-auto-gc**钩子会在垃圾回收开始之前被调用，可以用它来提醒你现在要回收垃圾了，或者依情形判断是否要中断回收。

### 3.3 服务器端钩子

除了客户端钩子，作为系统管理员，你还可以使用若干服务器端的钩子对项目强制执行各种类型的策略。这些钩子脚本在推送到服务器之前和之后运行。推送到服务器前运行的钩子可以在任何时候以非零值退出，拒绝推送并给客户端返回错误消息，还可以依你所想设置足够复杂的推送策略。

**pre-receive**

处理来自客户端的推送操作时，最先被调用的脚本是 pre-receive。它从标准输入获取一系列被推送的引用。如果它以非零值退出，所有的推送内容都不会被接受。你可以用这个钩子阻止对引用进行非快进（non-fast-forward）的更新，或者对该推送所修改的所有引用和文件进行访问控制。

**update**

update 脚本和 pre-receive 脚本十分类似，不同之处在于它会为每一个准备更新的分支各运行一次。假如推送者同时向多个分支推送内容，pre-receive 只运行一次，相比之下 update 则会为每一个被推送的分支各运行一次。它不会从标准输入读取内容，而是接受三个参数：引用的名字（分支），推送前的引用指向的内容的 SHA-1 值，以及用户准备推送的内容的 SHA-1 值。如果 update 脚本以非零值退出，只有相应的那一个引用会被拒绝；其余的依然会被更新。

**post-receive**

post-receive 挂钩在整个过程完结以后运行，可以用来更新其他系统服务或者通知用户。它接受与 pre-receive 相同的标准输入数据。它的用途包括给某个邮件列表发信，通知持续集成（continous integration）的服务器，或者更新问题追踪系统（ticket-tracking system）—— 甚至可以通过分析提交信息来决定某个问题（ticket）是否应该被开启，修改或者关闭。该脚本无法终止推送进程，不过客户端在它结束运行之前将保持连接状态，所以如果你想做其他操作需谨慎使用它，因为它将耗费你很长的一段时间。

## 4.强制策略

在本节中，你将应用前面学到的知识建立这样一个 Git 工作流程：检查提交信息的格式，并且指定只能由特定用户修改项目中特定的子目录。你将编写一个客户端脚本来提示开发人员他们的推送是否会被拒绝，以及一个服务器端脚本来实际执行这些策略。

我们待会展示的脚本是用 Ruby 写的，部分是由于我习惯用它写脚本，另外也因为 Ruby 简单易懂，即便你没写过它也能看明白。不过任何其他语言也一样适用。所有 Git 自带的示例钩子脚本都是用 Perl 或 Bash 写的，所以你能从它们中找到相当多的这两种语言的钩子示例。

### 4.1 服务器端钩子

所有服务器端的工作都将在你的 hooks 目录下的 update 脚本中完成。update 脚本会为每一个提交的分支各运行一次，它接受三个参数：

- 被推送的引用的名字
- 推送前分支的修订版本（revision）
- 用户准备推送的修订版本（revision）

如果推送是通过 SSH 进行的，还可以获知进行此次推送的用户的信息。如果你允许所有操作都通过公匙授权的单一帐号（比如“git”）进行，就有必要通过一个 shell 包装脚本依据公匙来判断用户的身份，并且相应地设定环境变量来表示该用户的身份。下面就假设`$USER`环境变量里存储了当前连接的用户的身份，你的 update 脚本首先搜集一切需要的信息：

```ruby
#!/usr/bin/env ruby

$refname = ARGV[0]
$oldrev  = ARGV[1]
$newrev  = ARGV[2]
$user    = ENV['USER']

puts "Enforcing Policies..."
puts "(#{$refname}) (#{$oldrev[0,6]}) (#{$newrev[0,6]})"
```

#### 1.指定特殊的提交信息格式

你的第一项任务是要求每一条提交信息都必须遵循某种特殊的格式。作为目标，假定每一条信息必须包含一条形似“ref: 1234”的字符串，因为你想把每一次提交对应到问题追踪系统（ticketing system）中的某个事项。你要逐一检查每一条推送上来的提交内容，看看提交信息是否包含这么一个字符串，然后，如果某个提交里不包含这个字符串，以非零返回值退出从而拒绝此次推送。

把`$newrev`和`$oldrev`变量的值传给一个叫做`git rev-list`的 Git 底层命令，你可以获取所有提交的SHA-1 值列表。`git rev-list`基本类似`git log`命令，但它默认只输出 SHA-1 值而已，没有其他信息。所以要获取由一次提交到另一次提交之间的所有 SHA-1 值，可以像这样运行：

```bash
$ git rev-list 538c33..d14fc7
d14fc7c847ab946ec39590d87783c69b031bdfb7
9f585da4401b0a3999e84113824d15245c13f0be
234071a1be950e2a8d078e6141f5cd20c1e61ad3
dfa04c9ef3d5197182f13fb5b9b1fb7717d2222a
17716ec0f1ff5c77eff40b7fe912f9f6cfd0e475
```

你可以截取这些输出内容，循环遍历其中每一个 SHA-1 值，找出与之对应的提交信息，然后用正则表达式来测试该信息包含的内容。

下一步要实现从每个提交中提取出提交信息。使用另一个叫做 git cat-file 的底层命令来获得原始的提交数据。我们将在【Git 内部原理】了解到这些底层命令的细节；现在暂时先看一下这条命令的输出：

```bash
$ git cat-file commit ca82a6
tree cfda3bf379e4f8dba8717dee55aab78aef7f4daf
parent 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
author Scott Chacon <schacon@gmail.com> 1205815931 -0700
committer Scott Chacon <schacon@gmail.com> 1240030591 -0700

changed the version number
```

通过 SHA-1 值获得提交中的提交信息的一个简单办法是找到提交的第一个空行，然后取从它往后的所有内容。可以使用 Unix 系统的`sed`命令来实现该效果：

```bash
$ git cat-file commit ca82a6 | sed '1,/^$/d'
changed the version number
```

你可以从每一个待推送的提交里提取提交信息，然后在提取的内容不符合要求时退出。为了退出脚本和拒绝此次推送，返回非零值。整个脚本大致如下：

```ruby
$regex = /\[ref: (\d+)\]/
# 指定自定义的提交信息格式
def check_message_format
  missed_revs = `git rev-list #{$oldrev}..#{$newrev}`.split("\n")
  missed_revs.each do |rev|
    message = `git cat-file commit #{rev} | sed '1,/^$/d'`
    if !$regex.match(message)
      puts "[POLICY] Your message is not formatted correctly"
      exit 1
    end
  end
end
check_message_format
```

把这一段放在 update 脚本里，所有包含不符合指定规则的提交都会遭到拒绝。

#### 2.指定基于用户的访问权限控制列表（ACL）系统

假设你需要添加一个使用访问权限控制列表的机制，来指定哪些用户对项目的哪些部分有推送权限。某些用户具有全部的访问权，其他人只对某些子目录或者特定的文件具有推送权限。为了实现这一点，你要把相关的规则写入位于服务器原始 Git 仓库的 acl 文件中。你还需要让 update 钩子检阅这些规则，审视推送的提交内容中被修改的所有文件，然后决定执行推送的用户是否对所有这些文件都有权限。

先从写一个 ACL 文件开始吧。这里使用的格式和 CVS 的 ACL 机制十分类似：它由若干行构成，第一项内容是 avail 或者 unavail，接着是逗号分隔的适用该规则的用户列表，最后一项是适用该规则的路径（该项空缺表示没有路径限制）。各项由管道符 | 隔开。

在本例中，你会有几个管理员，一些对 doc 目录具有权限的文档作者，以及一位仅对 lib 和 tests 目录具有权限的开发人员，相应的 ACL 文件如下：

```bash
avail|nickh,pjhyett,defunkt,tpw
avail|usinclair,cdickens,ebronte|doc
avail|schacon|lib
avail|schacon|tests
```

首先把这些数据读入你要用到的数据结构里。在本例中，为保持简洁，我们暂时只实现 avail 的规则。下面这个方法生成一个关联数组，它的键是用户名，值是一个由该用户有写权限的所有目录组成的数组：

```bash
def get_acl_access_data(acl_file)
  # 读取 ACL 数据
  acl_file = File.read(acl_file).split("\n").reject { |line| line == '' }
  access = {}
  acl_file.each do |line|
    avail, users, path = line.split('|')
    next unless avail == 'avail'
    users.split(',').each do |user|
      access[user] ||= []
      access[user] << path
    end
  end
  access
end
```

对于之前给出的 ACL 规则文件，这个 get_acl_access_data 方法返回的数据结构如下：

```bash
{"defunkt"=>[nil],
 "tpw"=>[nil],
 "nickh"=>[nil],
 "pjhyett"=>[nil],
 "schacon"=>["lib", "tests"],
 "cdickens"=>["doc"],
 "usinclair"=>["doc"],
 "ebronte"=>["doc"]}
```

既然拿到了用户权限的数据，接下来你需要找出提交都修改了哪些路径，从而才能保证推送者对所有这些路径都有权限。

使用`git log`的`--name-only`选项，我们可以轻而易举的找出一次提交里修改的文件：

```bash
$ git log -1 --name-only --pretty=format:'' 9f585d

README
lib/test.rb
```

使用 get_acl_access_data 返回的 ACL 结构来一一核对每次提交修改的文件列表，就能找出该用户是否有权限推送所有的提交内容：

```ruby
# 仅允许特定用户修改项目中的特定子目录
def check_directory_perms
  access = get_acl_access_data('acl')
  # 检查是否有人在向他没有权限的地方推送内容
  new_commits = `git rev-list #{$oldrev}..#{$newrev}`.split("\n")
  new_commits.each do |rev|
    files_modified = `git log -1 --name-only --pretty=format:''
#{rev}`.split("\n")
    files_modified.each do |path|
      next if path.size == 0
      has_file_access = false
      access[$user].each do |access_path|
        if !access_path  # 用户拥有完全访问权限
           || (path.start_with? access_path) # 或者对此路径有访问权限
          has_file_access = true
        end
      end
      if !has_file_access
        puts "[POLICY] You do not have access to push to #{path}"
        exit 1
      end
    end
  end
end

check_directory_perms
```

通过`git rev-list`获取推送到服务器的所有提交。接着，对于每一个提交，找出它修改的文件，然后确保推送者具有这些文件的推送权限。

现在你的用户没法推送带有不正确的提交信息的内容，也不能在准许他们访问范围之外的位置做出修改。

#### 3.测试一下

如果已经把上面的代码放到 .git/hooks/update 文件里了，运行`chmod u+x .git/hooks/update`，然后尝试推送一个不符合格式的提交，你会得到以下的提示：

```bash
$ git push -f origin master
Counting objects: 5, done.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 323 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
Unpacking objects: 100% (3/3), done.
Enforcing Policies...
(refs/heads/master) (8338c5) (c5b616)
[POLICY] Your message is not formatted correctly
error: hooks/update exited with error code 1
error: hook declined to update refs/heads/master
To git@gitserver:project.git
 ! [remote rejected] master -> master (hook declined)
error: failed to push some refs to 'git@gitserver:project.git'
```

这里有几个有趣的信息。首先，我们可以看到钩子运行的起点。

```bash
Enforcing Policies...
(refs/heads/master) (fb8c72) (c56860)
```

注意这是从 update 脚本开头输出到标准输出的。所有从脚本输出到标准输出的内容都会转发给客户端。

下一个值得注意的部分是错误信息。

```bash
[POLICY] Your message is not formatted correctly
error: hooks/update exited with error code 1
error: hook declined to update refs/heads/master
```

第一行是我们的脚本输出的，剩下两行是 Git 在告诉我们 update 脚本退出时返回了非零值因而推送遭到了拒绝。最后一点：

```bash
To git@gitserver:project.git
 ! [remote rejected] master -> master (hook declined)
error: failed to push some refs to 'git@gitserver:project.git'
```

你会看到每个被你的钩子拒之门外的引用都收到了一个 remote rejected 信息，它告诉你正是钩子无法成功运行导致了推送的拒绝。

又或者某人想修改一个自己不具备权限的文件然后推送了一个包含它的提交，他将看到类似的提示。比如，一个文档作者尝试推送一个修改到 lib 目录的提交，他会看到

```bash
[POLICY] You do not have access to push to lib/test.rb
```

从今以后，只要 update 脚本存在并且可执行，我们的版本库中永远都不会包含不符合格式的提交信息，并且用户都会待在沙箱里面。

### 4.2 客户端钩子

这种方法的缺点在于，用户推送的提交遭到拒绝后无法避免的抱怨。辛辛苦苦写成的代码在最后时刻惨遭拒绝是十分让人沮丧且具有迷惑性的；更可怜的是他们不得不修改提交历史来解决问题，这个方法并不能让每一个人满意。

逃离这种两难境地的法宝是给用户一些客户端的钩子，在他们犯错的时候给以警告。然后呢，用户们就能趁问题尚未变得更难修复，在提交前消除这个隐患。由于钩子本身不跟随克隆的项目副本分发，所以你必须通过其他途径把这些钩子分发到用户的 .git/hooks 目录并设为可执行文件。虽然你可以在相同或单独的项目里加入并分发这些钩子，但是 Git 不会自动替你设置它。

首先，你应该在每次提交前核查你的提交信息，这样才能确保服务器不会因为不合条件的提交信息而拒绝你的更改。为了达到这个目的，你可以增加 commit-msg 钩子。如果你使用该钩子来读取作为第一个参数传递的提交信息，然后与规定的格式作比较，你就可以使 Git 在提交信息格式不对的情况下拒绝提交。

```ruby
#!/usr/bin/env ruby
message_file = ARGV[0]
message = File.read(message_file)

$regex = /\[ref: (\d+)\]/

if !$regex.match(message)
  puts "[POLICY] Your message is not formatted correctly"
  exit 1
end
```

如果这个脚本位于正确的位置（.git/hooks/commit-msg）并且是可执行的，你提交信息的格式又是不正确的，你会看到：

```bash
$ git commit -am 'test'
[POLICY] Your message is not formatted correctly
```

在这个示例中，提交没有成功。然而如果你的提交注释信息是符合要求的，Git 会允许你提交：

```bash
$ git commit -am 'test [ref: 132]'
[master e05c914] test [ref: 132]
 1 file changed, 1 insertions(+), 0 deletions(-)
```

接下来我们要保证没有修改到 ACL 允许范围之外的文件。假如你的 .git 目录下有前面使用过的那份 ACL 文件，那么以下的 pre-commit 脚本将把里面的规定执行起来：

```ruby
#!/usr/bin/env ruby

$user    = ENV['USER']
# [ 插入上文中的 get_acl_access_data 方法 ]
# 仅允许特定用户修改项目中的特定子目录
def check_directory_perms
  access = get_acl_access_data('.git/acl')

  files_modified = `git diff-index --cached --name-only HEAD`.split("\n")
  files_modified.each do |path|
    next if path.size == 0
    has_file_access = false
    access[$user].each do |access_path|
    if !access_path || (path.index(access_path) == 0)
      has_file_access = true
    end
    if !has_file_access
      puts "[POLICY] You do not have access to push to #{path}"
      exit 1
    end
  end
end

check_directory_perms
```

这和服务器端的脚本几乎一样，除了两个重要区别。第一，ACL 文件的位置不同，因为这个脚本在当前工作目录运行，而非 .git 目录。ACL 文件的路径必须从

```ruby
access = get_acl_access_data('acl')
```

修改成：

```ruby
access = get_acl_access_data('.git/acl')
```

另一个重要区别是获取被修改文件列表的方式。在服务器端的时候使用了查看提交纪录的方式，可是目前的提交都还没被记录下来呢，所以这个列表只能从暂存区域获取。和原来的

```ruby
files_modified = `git log -1 --name-only --pretty=format:'' #{ref}`
```

不同，现在要用

```ruby
files_modified = `git diff-index --cached --name-only HEAD`
```

不同的就只有这两个——除此之外，该脚本完全相同。有一点要注意的是，它假定在本地运行的用户和推送到远程服务器端的相同。如果这二者不一样，则需要手动设置一下`$user`变量。

在这里，我们还可以确保推送内容中不包含非快进（non-fast-forward）的引用。出现一个不是快进（fast-forward）的引用有两种情形，要么是在某个已经推送过的提交上作变基，要么是从本地推送一个错误的分支到远程分支上。

假定为了执行这个策略，你已经在服务器上配置好了 receive.denyDeletes 和 receive.denyNonFastForwards，因而唯一还需要避免的是在某个已经推送过的提交上作变基。

下面是一个检查这个问题的 pre-rebase 脚本示例。它获取所有待重写的提交的列表，然后检查它们是否存在于远程引用中。一旦发现其中一个提交是在某个远程引用中可达的（reachable），它就终止此次变基：

```ruby
#!/usr/bin/env ruby

base_branch = ARGV[0]
if ARGV[1]
  topic_branch = ARGV[1]
else
  topic_branch = "HEAD"
end

target_shas = `git rev-list #{base_branch}..#{topic_branch}`.split("\n")
remote_refs = `git branch -r`.split("\n").map { |r| r.strip }

target_shas.each do |sha|
  remote_refs.each do |remote_ref|
    shas_pushed = `git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}`
    if shas_pushed.split("\n").include?(sha)
      puts "[POLICY] Commit #{sha} has already been pushed to
#{remote_ref}"
      exit 1
    end
  end
end
```

此脚本使用了不曾提到的语法。通过运行这个命令可以获得一系列之前推送过的提交：

```ruby
`git rev-list ^#{sha}^@ refs/remotes/#{remote_ref}`
```

`SHA^@`语法会被解析成该提交的所有父提交。该命令会列出在远程分支最新的提交中可达的，却在所有我们尝试推送的提交的 SHA-1 值的所有父提交中不可达的提交——也就是快进的提交。

这个解决方案主要的问题在于它有可能很慢而且常常没有必要——只要你不用`-f`来强制推送，服务器就会自动给出警告并且拒绝接受推送。然而，这是个不错的练习，而且理论上能帮助你避免一次以后可能不得不回头修补的变基。
