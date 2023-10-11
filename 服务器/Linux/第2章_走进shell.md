# 第2章_走进shell

> **本章内容**
>
> - 访问命令行
> - 通过 Linux 控制台终端访问 CLI 及通过图形化终端仿真器访问 CLI
> - 使用 GNOME Terminal 终端仿真器
> - 使用 Konsole 终端仿真器
> - 使用 xterm 终端仿真器

在 Linux 早期，系统管理员、程序员、系统用户全都端坐在 Linux 控制台终端前，输入 shell 命令，查看文本输出。如今伴随着图形化桌面环境的应用，想在系统中找到 shell 提示符来输入命令都变得困难起来。本章讨论了如何进入命令行环境，并带你逐步了解可能会在各种 Linux 发行版中碰到的终端仿真软件包。

## 1.进入命令行

在图形化桌面出现之前，和 Unix 系统交互的唯一方式就是通过 shell 提供的文本**命令行界面**（command line interface，CLI）。CLI 只允许输入文本，而且只能显示文本和基本图形输出。

由于此限制，输出设备也用不着多高级，只需要一个简单的哑终端就能和 Unix 系统交互了。哑终端（dumb terminal）是由通信电缆（通常是多线束串行电缆，也叫带状电缆）连接到 Unix 系统的显示器和键盘。通过这种简单的组合，可以轻松地向 Unix 系统输入文本数据并显示文本结果。

你也很清楚，如今的 Linux 环境已经大不同往日了。大部分 Linux 发行版采用了某种类型的图形化桌面环境。但要输入 shell 命令，仍然需要通过文本显示来访问 shell 的 CLI。于是现在的问题归结为一点：有时候在 Linux 发行版中找到进入 CLI 的途径还真不是件容易的事。

### 1.1 控制台终端

进入 CLI 的一种途径是访问 Linux 系统的文本模式。该模式只在显示器上提供一个简单的 shell CLI，就跟图形化桌面出现之前那样。这称作**Linux控制台**，因为它模拟的是早期的硬接线控制台终端（hard-wired console terminal），而且是跟 Linux 系统交互的直接接口。

Linux 系统启动时，会自动创建多个**虚拟控制台**。虚拟控制台是运行在 Linux 系统内存中的终端会话。多数 Linux 发行版会启动 5~6 个（甚至更多）虚拟控制台代替哑终端，通过单个计算机键盘和显示器就可以访问这些虚拟控制台。

### 1.2 图形化终端

虚拟控制台终端的另一种替代方案是使用 Linux 图形化桌面环境中的**终端仿真软件包**。终端仿真软件包会在桌面图形化窗口中模拟控制台终端。下图显示了一个运行在 Linux 图形化桌面环境中的终端仿真器。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/008-341239707e7630c822d488db02f9128f-8a6096.jpg" alt="{%}" style="zoom:80%;" />

图形化终端仿真只负责 Linux 图形化体验的一部分。完整的体验需要借助包括图形化终端仿真软件（称为**客户端**）在内的多个组件来实现。下表显示了 Linux 图形化桌面环境中不同的组件。

| 名称       | 例子                                                         | 描述                                                   |
| :--------- | :----------------------------------------------------------- | :----------------------------------------------------- |
| 客户端     | 图形化终端仿真器，桌面环境（GNOME Shell、KDE Plasma），网络浏览器 | 请求图形化服务的应用程序                               |
| 显示服务器 | Wayland、X Window System                                     | 管理显示（屏幕）和输入设备（键盘、鼠标、触摸屏）的元素 |
| 窗口管理器 | Mutter、Metacity、Kwin                                       | 为窗口添加边框并提供窗口移动和管理功能的元素           |
| 小部件库   | Plasmoids、Cinnamon Spices                                   | 为桌面环境客户端添加菜单和外观项的元素                 |

要想在桌面中使用命令行，关键在于图形化终端仿真器。你可以将图形化终端仿真器看作图形化用户界面中（in the GUI）的 CLI 终端，将虚拟控制台终端看作图形化用户界面之外（outside the GUI）的 CLI 终端。理解各种终端及其特性能够提高你的命令行体验。

## 2.通过Linux控制台终端访问CLI

在 Linux 早期，引导系统时你在显示器上只能看到一个登录提示符，除此之外就没别的了。之前说过，这就是 Linux 控制台。它是可以向系统输入命令的唯一地方。

尽管在引导时会创建多个虚拟控制台，但很多 Linux 发行版在完成启动过程之后会切换到图形化环境中。这为用户提供了图形化登录以及桌面体验。对于这类系统，就只能通过手动方式来访问虚拟控制台了。

在大多数 Linux 发行版中，可以使用简单的按键组合来访问某个 Linux 虚拟控制台。通常必须按下 Ctrl+Alt 组合键，然后再按一个功能键（F1～F7）来进入你要使用的虚拟控制台。功能键 F2 键会生成虚拟控制台 2，F3 键会生成虚拟控制台 3，F4 键会生成虚拟控制台 4，以此类推。

> **注意**
>
> Linux 发行版通常使用 Ctrl+Alt 组合键配合 F1 键、F7 键或 F8 键进入图形化界面。Ubuntu 和 CentOS 均使用 F1 键。不过最好还是自己动手测试一下，看看你用的发行版是如何分配按键的，尤其是对于比较旧的发行版。

文本模式的虚拟控制台采用全屏的方式显示文本登录界面。下图展示了一个虚拟控制台的文本登录界面。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/009-acea1c2b36b25de722b09f34b4afb665-1a115e.jpg" alt="{%}" style="zoom:80%;" />

注意第一行文本的最后一个单词`tty2`，其中的`2`表明这是虚拟控制台 2，可以通过按下 Ctrl+Alt+F2 组合键进入。`tty`代表**电传打字机**（teletypewriter）。这个词有些年代了，是一种用于发送消息的机器。

> **注意**
>
> 不是所有的 Linux 发行版都会在登录画面显示虚拟控制台的`tty`编号。登入虚拟控制台后，可以输入命令`tty`，然后按 Enter 键查看当前使用的是哪个虚拟控制台。第 3 章会介绍命令输入。

在`login:`提示符后输入你的用户 ID，然后在`Password:`提示符后输入密码就可以登入控制台终端了。如果你之前从来没有用过这种登录方式，则要注意在这里输入的密码和在图形化环境中输入的看起来不太一样。在图形化环境中，在你输入密码的时候会看到点号或者星号。但是在虚拟控制台中，输入密码的时候**什么都不会显示**。

> **注意**
>
> 记住，在 Linux 虚拟控制台中是无法运行任何图形化程序的。

登入虚拟控制台之后，就进入了 Linux CLI，你可以在不中断当前活动会话的情况下切换到另一个虚拟控制台，在所有的虚拟控制台之间任意切换，同时拥有多个活动会话。在使用 CLI 时，这个特性提供了巨大的灵活性。

其他灵活性来自虚拟控制台的外观。尽管虚拟控制台只是一个文本模式的控制台终端，但你也可以修改文字和背景色。

例如，可以将终端的背景色设置成白色，将文本设置成黑色，这样可以让你的眼睛轻松些。登录之后，有好几种方法可以实现这种改动。一种方法是输入命令`setterm --inversescreen on`，然后按 Enter 键，如下图所示。注意，下图中使用`on`启用了`--inversescreen`特性。也可以使用`off`关闭该特性。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/010-131836040a8d212ee9d0667809b755f6-4d6f91.jpg" alt="{%}" style="zoom:80%;" />

另一种方法是先后输入两个命令。首先输入`setterm --background white`，然后按 Enter 键，接着输入`setterm -foreground black`，再按 Enter 键。要注意，因为先修改的是终端的背景色，所以可能不容易看清楚接下来输入的命令。

在上面的命令中，不用像`--inversescreen`那样去启用或关闭什么特性。共有 8 种颜色可供选择，分别是`black`、`red`、`green`、`yellow`、`blue`、`magenta`、`cyan`和`white`（`white`在有些发行版中看起来像是灰色）。你可以赋予纯文本模式的控制台终端富有创意的外观效果。下表展示了`setterm`命令的部分选项，可以用于改善控制台终端的可读性或外观。

| 选项              | 参数                                                         | 描述                                              |
| :---------------- | :----------------------------------------------------------- | :------------------------------------------------ |
| `--background`    | `black`、`red`、`green`、`yellow`、`blue`、`magenta`、`cyan`或`white` | 将终端的背景色改为指定颜色                        |
| `--foreground`    | `black`、`red`、`green`、`yellow`、`blue`、`magenta`、`cyan`或`white` | 将终端的前景色（特别是文本）改为指定颜色          |
| `--inversescreen` | `on`或`off`                                                  | 交换背景色和前景色                                |
| `--reset`         | 无                                                           | 将终端外观恢复成默认设置并清屏                    |
| `--store`         | 无                                                           | 将终端当前的前景色和背景色设置成`--reset`选项的值 |

如果不涉及 GUI，那么使用虚拟控制台终端访问 CLI 自然是一种不错的选择。但有时候你需要一边访问 CLI，一边运行图形化程序。使用终端仿真软件包可以解决这个问题，这也是在 GUI 中访问 shell CLI 的一种流行的方式。接下来几节会介绍提供图形化终端仿真的常见软件包。

## 3.通过图形化终端仿真器访问CLI

相较于虚拟控制台终端，图形化桌面环境提供了多种方式来访问 CLI。在图形化环境下，有大量可用的终端仿真器。每个软件包都有各自独特的特性以及选项。下表中列举出了一些流行的图形化终端仿真器软件包及其网站。

| 名称            | 网站                                                         |
| :-------------- | :----------------------------------------------------------- |
| Alacritty       | [github.com/alacritty/alacritty](http://github.com/alacritty/alacritty) |
| cool-retro-term | [github.com/Swordfish90/cool-retro-term](http://github.com/Swordfish90/cool-retro-term) |
| GNOME Terminal  | [wiki.gnome.org/Apps/Terminal](http://wiki.gnome.org/Apps/Terminal) |
| Guake           | [guake-project.org](http://guake-project.org/)               |
| Konsole         | [konsole.kde.org](http://konsole.kde.org/)                   |
| kitty           | [sw.kovidgoyal.net/kitty](http://sw.kovidgoyal.net/kitty)    |
| rxvt-unicode    | [software.schmorp.de/pkg/rxvt-unicode.html](http://software.schmorp.de/pkg/rxvt-unicode.html) |
| Sakura          | [pleyades.net/david/projects/sakura](http://pleyades.net/david/projects/sakura) |
| St              | [st.suckless.org](http://st.suckless.org/)                   |
| Terminator      | [gnometerminator.blogspot.com](http://gnometerminator.blogspot.com/) |
| Terminology     | [enlightenment.org/about-terminology.md](http://enlightenment.org/about-terminology.md) |
| Termite         | [github.com/thestinger/termite](http://github.com/thestinger/termite) |
| Tilda           | [github.com/lanoxx/tilda](http://github.com/lanoxx/tilda)    |
| xterm           | [invisible-island.net/xterm](http://invisible-island.net/xterm) |
| Xfce4-terminal  | [docs.xfce.org/apps/terminal/start](http://docs.xfce.org/apps/terminal/start) |
| Yakuake         | [kde.org/applications/system/org.kde.yakuake](http://kde.org/applications/system/org.kde.yakuake) |

虽然有不少图形化终端仿真器软件包可用，但本章仅关注 3 个：GNOME Terminal、Konsole 和 xterm，不同的 Linux 发行版会选择其中之一作为默认安装。

## 4.使用GNOME Terminal终端仿真器

GNOME Terminal 是 GNOME Shell 桌面环境的默认终端仿真器。包括 Red Hat Enterprise Linux（RHEL）、CentOS 和 Ubuntu 在内的很多发行版默认采用 GNOME Shell 桌面环境，自然也默认使用 GNOME Terminal。GNOME Terminal 易于上手，是 Linux 新手不错的选择。本节将带你学习如何访问、配置和使用 GNOME Terminal。

### 4.1 访问GNOME Terminal

在 GNOME Shell 桌面环境中，访问 GNOME Terminal 很简单。点击桌面左上角的 Activities 图标。出现搜索栏时，在其中输入 terminal。如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/011-1024a265f9ea988cf290086310944ede-29c280.jpg" alt="{%}" style="zoom:80%;" />

注意，GNOME Terminal 应用程序图标的名字是 Terminal。点击图标就可以打开终端仿真器。在 CentOS 发行版中打开的 GNOME Terminal 如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/012-0a0d898d9cdc44f6b8b7c19c81853d23-6c6229.jpg" alt="{%}" style="zoom:80%;" />

使用完终端仿真器后，和其他桌面窗口一样，点击窗口右上角的 x 就可以将其关闭。

GNOME Terminal 的外观可能会随 Linux 发行版而有所不同。例如，下图展示了 Ubuntu GNOME Shell 桌面环境中的 GNOME Terminal。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/013-890fb7cb0ee1fb69a9b43e72f0f346cc-5449d4.jpg" alt="{%}" style="zoom:80%;" />

注意，以上两个 GNOME Terminal 的外观不一样。这通常是由于应用程序的默认配置（本章随后会介绍）以及 Linux 发行版的 GUI 窗口的不同特性造成的。

> **提示**
>
> 如果你使用的不是 GNOME Shell 桌面环境（安装了 GNMOE Ternimal），那么有可能并没有搜索功能。在这种情况下，可以使用桌面环境的菜单系统来查找 GNOME Terminal。一般来说，名称是 Terminal。

在很多发行版中，当你第一次运行 GNOME Terminal 时，终端仿真器图标会出现在 GNOME Shell Favorites 工具栏内。将鼠标悬停在该图标之上就会显示出终端仿真器的名称，如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/014-53f516f8091423c7184bf2baa4140ba3-0165a7.jpg" alt="{%}" style="zoom:80%;" />

如果图标没有出现在 Favorites 工具栏内，则可以设置快捷键来运行 GNOME Terminal。这种方法对于那些不喜欢使用鼠标的用户来说很方便，可以更快地访问 CLI。

> **提示**
>
> Ubuntu 发行版中的 GNOME Shell 已经创建好了打开 GNOME Terminal 的快捷键：`Ctrl+Alt+T`。

要想创建快捷键，需要访问 Keyboard Settings 中的 Keyboard Shortcuts 窗口。为了快速完成设置，点击 GNOME Shell 桌面左上角的 Activities 图标。当出现搜索栏时，点击搜索栏，在其中输入 Keyboard Shortcuts。之后的结果如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/015-d0b4d8a9394015fdc5a8954258afeb71-91a23b.jpg" alt="{%}" style="zoom:80%;" />

打开 Keyboard Shortcuts 窗口之后，使用鼠标向下滚动到窗口底部的 + 按钮。点击该按钮，打开对话框，可以在其中命名新的快捷方式，提供用于打开应用程序的命令，并设置该快捷方式的组合键，如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/016-38904c81e05e7006541e0e120f5fbf9e-3438c2.jpg" alt="{%}" style="zoom:80%;" />

要想顺利运行 GNOME Terminal，重要的是要使用正确的命令名，所以要在 Command 字段中输入`gnome-terminal`，如上图所示。一切设置妥当之后，点击 Add 按钮。现在就可以使用指定的快捷键快速启动 GNOME Terminal 了。

GNOME Terminal 通过菜单和快捷键提供了一些配置选项，可以在启动 GNOME Terminal 之后应用。了解这些选项可以提高 GNOME Terminal CLI 的体验。

### 4.2 菜单栏

GNOME Terminal 的菜单栏包含配置选项和定制选项，你可以通过这些选项打造符合自己使用习惯的 GNOME Terminal。

> **提示**
>
> 如果 GNOME Terminal 窗口没有显示菜单栏，那么用鼠标右键单击终端仿真器会话区域，在弹出的菜单中选择 Show Menubar。

下展示了 GNOME Terminal 的 File 菜单下的配置选项。File 菜单中包含了可用于创建和管理所有 CLI 终端会话的菜单项。

**File菜单**

| 名称         | 快捷键       | 描述                                                         |
| :----------- | :----------- | :----------------------------------------------------------- |
| New Tab      | Shift+Ctrl+T | 在现有的 GNOME Terminal 窗口的新标签中启动一个新的 shell 会话 |
| New Window   | Shift+Ctrl+N | 在新的 GNOME Terminal 窗口中启动一个新的 shell 会话          |
| Close Tab    | Shift+Ctrl+W | 在 GNOME Terminal 窗口中关闭当前标签                         |
| Close Window | Shift+Ctrl+Q | 关闭当前 GNOME Terminal 窗口                                 |

注意，就像在网络浏览器中一样，你可以在 GNOME Terminal 会话中打开新标签，启动全新的 CLI 会话。每个标签会话均被视为独立的 CLI 会话。

> **提示**
>
> 并不是必须通过点击菜单项才能访问 File 菜单中的选项。在终端模拟器会话区域中右键单击，也可以使用部分 File 菜单选项。

下表所示的 Eidt 菜单中包含用于处理标签内文本内容的菜单项。你可以在会话窗口中的任意位置复制和粘贴文本。

**Edit菜单**

| 名称         | 快捷键       | 描述                                            |
| :----------- | :----------- | :---------------------------------------------- |
| Copy         | Shift+Ctrl+C | 将所选文本复制到 GNOME 的剪贴板中               |
| Copy as HTML | None         | 将所选文本及其字体和颜色复制到 GNOME 的剪贴板中 |
| Paste        | Shift+Ctrl+V | 将 GNOME 剪贴板中的文本粘贴到会话中             |
| Select All   | None         | 选中回滚缓冲区（scrollback buffer）中的全部输出 |
| Preferences  | None         | 编辑当前会话的配置文件                          |

如果你缺乏键盘操作技能，则在终端中复制和粘贴命令非常有用。因此，GNOME Terminal 的 Copy 和 Paste 功能的键盘快捷键值得记下来。

> **注意**
>
> 在查看GNOME Terminal菜单选项时，记住，你所用的Linux发行版的GNOME Terminal的可用选择也许有所不同。这是因为有些Linux发行版使用的GNOME Terminal版本较旧。可以点击Help菜单，选择其中的About菜单项来查看版本号。

下表所示的 View 菜单中包含用于控制 CLI 会话窗口外观的菜单项。这些选项能够给视力有缺陷的用户带来帮助。

**View菜单**

| 名称         | 快捷键 | 描述                              |
| :----------- | :----- | :-------------------------------- |
| Show Menubar | None   | 打开 / 关闭菜单栏                 |
| Full Screen  | F11    | 打开 / 关闭终端窗口全桌面显示模式 |
| Zoom In      | Ctrl++ | 逐步增大窗口中的文本字号          |
| Normal Size  | Ctrl+0 | 恢复默认字号                      |
| Zoom Out     | Ctrl+- | 逐步减小窗口中的文本字号          |

注意，如果关闭了菜单栏显示，那么会话的菜单栏就会消失。不过，可以在任何一个终端会话窗口中单击右键，然后选择 Show Menubar，轻而易举地找回菜单栏。

下表所展示的 Serach 菜单中的菜单项用于在终端会话中进行简单的搜索。这些搜索与你在网络浏览器或文字处理软件中进行的操作类似。

**Search菜单**

| 名称            | 快捷键       | 描述                                     |
| :-------------- | :----------- | :--------------------------------------- |
| Find            | Shift+Ctrl+F | 打开 Find 窗口，指定待搜索的文本         |
| Find Next       | Shift+Ctrl+G | 从终端会话的当前位置开始向前搜索指定文本 |
| Find Previous   | Shift+Ctrl+H | 从终端会话的当前位置开始向后搜索指定文本 |
| Clear Highlight | Shift+Ctrl+J | 去除已查找到文本的高亮显示               |

下表所示的 Terminal 菜单中包含用于控制终端仿真会话特性的菜单项。这些菜单项并没有对应的快捷键。

**Terminal菜单**

| 名称            | 描述                                                      |
| :-------------- | :-------------------------------------------------------- |
| Read-Only       | 允许 / 禁止终端会话接受键盘输入，该菜单项并不会影响快捷键 |
| Reset           | 发出重置终端会话控制码                                    |
| Reset and Clear | 发出重置终端会话控制码并清除终端会话显示                  |
| 80x24           | 将当前终端窗口尺寸调整为80列宽x24行高                     |
| 80x43           | 将当前终端窗口尺寸调整为80列宽x43行高                     |
| 132x24          | 将当前终端窗口尺寸调整为132列宽x24行高                    |
| 130x43          | 将当前终端窗口尺寸调整为130列宽x43行高                    |

Reset 菜单项极其有用。你有时候可能意外地导致终端会话显示了一堆杂乱无章的字符和符号。这时候根本分辨不出文本信息。这通常是由于在屏幕上显示了非文本文件。可以通过选择 Reset 或 Reset and Clear 让终端会话恢复正常。

> **注意**
>
> 记住，在调整终端窗口尺寸时（比如使用 Terminal 菜单中的80列宽x24行高设置），实际的窗口大小受制于所用的字体。最好的办法是尝试不同的设置，从中找出适合的尺寸。

下表所示的 Tabs 菜单包含用于控制标签位置以及活动标签选择的菜单项。这个菜单只有当你打开了多个标签会话时才会出现。

**Tabs菜单**

| 名称                | 快捷键               | 描述                                                    |
| :------------------ | :------------------- | :------------------------------------------------------ |
| Previous Tab        | Ctrl+Page Up         | 使上一个标签成为活动标签                                |
| Next Tab            | Ctrl+Page Down       | 使下一个标签成为活动标签                                |
| Move Terminal Left  | Shift+Ctrl+Page Up   | 将当前标签放在前一个标签之前                            |
| Move Terminal Right | Shift+Ctrl+Page Down | 将当前标签放在前一个标签之后                            |
| Detach Terminal     | None                 | 删除选项卡并使用此选项卡会话启动一个新的 GNOME 终端窗口 |

最后，Help 菜单包含两个菜单项。

- Contents 提供了完整的 GNOME Terminal 手册，你可以从中研究 GNOME Terminal 的各个菜单项和特性
- About 显示了当前正在运行的 GNOME Terminal 的版本

除了 GNOME Terminal 终端仿真软件包，另一个常用的软件包是 Konsole。尽管两者在很多方面类似，但还是存在着相当大的差异，我们有必要单独开辟一节来讲解。

## 5.使用Konsole终端仿真器

KDE 项目拥有自己的终端仿真软件包 Konsole。Konsole 具备基本的终端仿真特性，还提供了更高级的图形应用程序功能。本节描述了 Konsole 的各种特性及其用法。

### 5.1 访问Konsole终端仿真器

Konsole 是 KDE 桌面环境 Plasma 默认的终端仿真器，可以轻松地通过 KDE 环境的菜单系统访问。在其他桌面环境中，通常要利用搜索功能访问 Konsole。

在 KDE 桌面环境（Plasma）中，要想启动 Konsole 终端仿真器，可以点击屏幕左下方的 Application Launcher 图标，然后点击 Applications ➪ System ➪ Terminal (Konsole)。

> **注意**
>
> 在 Plasma 菜单环境中，你可能会看到两个或更多的终端菜单项。如果是这样，则下方带有文字 Konsole 的 Terminal 菜单项就是 Konsole 终端仿真器。

在 GNOME Shell 桌面环境中，通常默认并未安装 Konsole。如果安装了 Konsole，可以通过 GNOME Shell 的搜索功能访问。点击桌面左上角的 Activities 图标。出现搜索栏时，点击搜索栏，在其中输入 konsole。如果系统中的终端仿真器可用，你就会看到出现 Konsole 的图标。

> **注意**
>
> 你的系统中可能没有安装 Konsole 终端仿真软件包。如果想安装的话，请阅读第 9 章来学习如何在命令行中安装软件。

点击 Konsole 图标，打开终端仿真器。在 Ubuntu 发行版中打开的 Konsole 如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/017-78b318c83eb03555987df42c472e788f-489866.jpg" alt="{%}" style="zoom:80%;" />

别忘了在大多数桌面环境中可以创建快捷键来访问 Konsole 等应用程序。启动 Konsole 终端仿真器要用到的命令是`konsole`。如果已经安装过 Konsole，则可以在其他的终端仿真器中输入 konsole，然后按 Enter 键来启动它。

> **提示**
>
> 在 Plasma 桌面环境中已经为 Konsole 终端仿真器设置好了默认快捷键：Ctrl+Alt+T。
>
> （与在 GNOME Shell 中打开 GNOME Terminal 的快捷键相同）

与 GNOME Terminal 类似，Konsole 终端仿真器也通过菜单和快捷键提供了多个配置选项。接下来会逐一讲解。

### 5.2 菜单栏

Konsole 的菜单栏包含了查看和更改终端仿真会话特性所需的配置及定制化选项。

> **提示**
>
> 如果没有看到 Konsole 菜单栏，可以按 Ctrl+Shift+M 组合键将其显示出来。

下表所示的 File 菜单包含用于在当前窗口或新窗口中启动新标签的菜单项。

**File菜单**

| 名称              | 快捷键       | 描述                                                         |
| :---------------- | :----------- | :----------------------------------------------------------- |
| New Window        | Ctrl+Shift+N | 在新的 Konsole Terminal 窗口中启动一个新的 shell 会话        |
| New Tab           | Ctrl+Shift+T | 在现有 Konsole Terminal 窗口的新标签中启动一个新的 shell 会话 |
| Clone Tab         | None         | 在现有 Konsole Terminal 窗口的新选项卡中启动一个新的 shell 会话并尝试复制当前标签 |
| Save Output As    | Ctrl+Shift+S | 将回滚缓冲区中当前标签的输出保存为文本文件或 HTML 文件       |
| Print Screen      | Ctrl+Shift+P | 打印当前标签的显示内容                                       |
| Open File Manager | None         | 打开默认的文件浏览器                                         |
| Close Session     | Ctrl+Shift+W | 关闭当前标签会话                                             |
| Close Window      | Ctrl+Shift+Q | 关闭当前 Konsole 窗口                                        |

Konsole 提供了两个方便的菜单项来保存 shell 会话信息：Save Output As 和 Print Screen。Print Screen 允许使用系统打印机来打印当前标签的显示内容或将其保存为 PDF 文件。

> **注意**
>
> 在阅读这些 Konsole 菜单项时，记住，你所使用的 Linux 发行版中的 Konsole 提供的菜单项可能和在这里看到的大不相同。这是因为一些 Linux 发行版安装的依然是比较旧的 Konsole 终端仿真软件包。

下表中所示的 Edit 菜单包含用于处理会话文本内容的菜单项。除此之外，还可以管理标签名称。

**Edit菜单**

| 名称          | 快捷键       | 描述                                                   |
| :------------ | :----------- | :----------------------------------------------------- |
| Copy          | Ctrl+Shift+C | 将所选择的文本复制到Konsole的剪贴板中                  |
| Paste         | Ctrl+Shift+V | 将Konsole剪贴板中的文本粘贴到会话中                    |
| Select All    | None         | 选中当前标签中的所有文本                               |
| Copy Input To | None         | 开始 / 停止将会话输入复制到选定的其他会话中            |
| Send Signal   | None         | 将选定的信号发送至当前标签的shell进程或其他进程        |
| Rename Tab    | Ctrl+Alt+S   | 修改会话标签的标题                                     |
| ZModem Upload | Ctrl+Alt+U   | 开始上传所选中的文件（如果支持ZMODEM文件传输协议的话） |
| Find          | Ctrl+Shift+F | 打开Find窗口，提供回滚缓冲区文本搜索选项               |
| Find Next     | F3           | 在回滚缓冲区历史中查找下一处文本匹配                   |
| Find Previous | Shift+F3     | 在回滚缓冲区历史中查找上一处文本匹配                   |

Konsole 提供了一种不错的方法来跟踪每个标签会话的用途。可以使用 Rename Tab 菜单项为标签起一个符合其用途的名称。这有助于分辨打开的标签会话究竟是用来做什么的。

> **注意**
>
> Konsole 维护着每个标签的历史记录（以前称为**回滚缓冲区**）。历史记录包含了终端查看区域的输出文本。在默认情况下，保留回滚缓冲区中的最后 1000 行。只需使用查看区域中的滚动条即可在回滚缓冲区中回滚。还可以按Shift+向上箭头键逐行向后滚动或按Shift+PageUp键一次向后滚动一页（24行）。

下表所示的 View 菜单包含用于控制 Konsole 窗口中单个会话视图的菜单项。除此之外还可以监视终端会话活动。

**View菜单**

| 名称                       | 快捷键       | 描述                                                        |
| :------------------------- | :----------- | :---------------------------------------------------------- |
| Split View                 | None         | 控制显示在当前 Konsole 窗口中的多个标签会话                 |
| Detach Current Tab         | Ctrl+Shift+L | 删除一个标签会话，并使用该标签会话启动一个新的 Konsole 窗口 |
| Detach Current View        | Ctrl+Shift+H | 删除当前标签会话的视图，并使用它启动一个新的 Konsole 窗口   |
| Monitor for Silence        | Ctrl+Shift+I | 打开 / 关闭无活动标签（tab silence）的特殊消息              |
| Monitor for Activity       | Ctrl+Shift+A | 打开 / 关闭活动标签（tab activity）的特殊消息               |
| Read-only                  | None         | 允许 / 禁止终端会话接受键盘输入，不影响键盘快捷键           |
| Enlarge Font               | Ctrl++       | 逐步增大窗口中的文本字号                                    |
| Reset Font Size            | Ctrl+Alt+0   | 重置文本字号                                                |
| Shrink Font                | Ctrl+-       | 逐步减小窗口中的文本字号                                    |
| Set Encoding               | None         | 设置用于发送和显示字符的字符集                              |
| Clear Scrollback           | None         | 删除当前会话的回滚缓冲区中的文本                            |
| Clear Scrollback and Reset | Ctrl+Shift+K | 删除当前会话的回滚缓冲区中的文本并重置终端窗口              |
| Full Screen Mode           | F11          | 打开 / 关闭终端窗口的全屏显示模式                           |

菜单项 Monitor for Silence 用于指明无活动标签。如果在当前标签会话内超过 7 秒没有出现新内容，则该标签即为无活动标签。这允许你在等待应用程序输出的时候切换到另一个标签。

> **提示**
>
> 当你在活动会话区域单击鼠标右键时，Konsole 会弹出一个简单的菜单，其中包含一些菜单项。

下表所示的 Bookmarks 菜单中的菜单项可用于管理 Konsole 窗口的**书签**。书签能够保存活动会话的目录位置，随后可以在相同会话或新的会话中返回到这些位置。

**Bookmarks菜单**

| 名称                    | 快捷键       | 描述                                   |
| :---------------------- | :----------- | :------------------------------------- |
| Add Bookmark            | Ctrl+Shift+B | 在当前目录位置创建一个新书签           |
| Bookmark Tabs as Folder | None         | 为当前所有的终端标签会话创建一个新书签 |
| New Bookmark Folder     | None         | 创建一个新的书签文件夹                 |
| Edit Bookmarks          | None         | 编辑已有的书签                         |

下表所示的 Settings 菜单包含可用于定制和管理配置文件的菜单项。配置文件允许用户自动运行命令、设置会话外观、配置回滚缓冲区等。你还可以通过 Setting 菜单为 shell 会话多添加一点儿功能。

**Setting菜单**

| 名称                         | 快捷键       | 描述                                            |
| :--------------------------- | :----------- | :---------------------------------------------- |
| Edit Current Profile         | None         | 打开 Edit Profile 窗口，提供配置文件配置选项    |
| Switch Profile               | None         | 将所选的配置文件应用于当前标签                  |
| Manage Profiles              | None         | 打开 Manage Profiles 窗口，提供配置文件管理选项 |
| Show Menubar                 | Ctrl+Shift+M | 打开 / 关闭菜单栏显示                           |
| Configure Keyboard Shortcuts | None         | 创建 Konsole 命令键盘快捷键                     |
| Configure Notifications      | None         | 创建自定义的 Konsole 提醒                       |
| Configure Konsole            | Ctrl+Shift+, | 配置很多 Konsole 特性                           |

Configure Notifications 允许将会话中发生的特定事件与不同的行为关联起来，比如播放声音。当出现某个事件时，就会触发指定的行为（或一系列行为）。

下表所示的 Help 菜单提供了完整的 Konsole 手册（如果你的 Linux 发行版中已经安装了 KDE 手册的话）以及标准的 About Konsole 对话框。

**Help菜单**

| 名称                        | 快捷键   | 描述                                      |
| :-------------------------- | :------- | :---------------------------------------- |
| Konsole Handbook            | None     | 包含完整的 Konsole 手册                   |
| What's This?                | Shift+F1 | 包含终端部件（terminal widget）的帮助信息 |
| Report Bug                  | None     | 打开 Submit BugReport 表单                |
| Donate                      | None     | 在 Web 浏览器中打开 KDE 捐赠页面          |
| Switch Application Language | None     | 打开 Switch Application Language 表单     |
| About Konsole               | None     | 显示包括 Konsole 当前版本在内的相关信息   |
| About KDE                   | None     | 显示 KDE 桌面环境的相关信息               |

Help 菜单提供了一份全面详实的文档以帮助你使用 Konsole。除此之外，在你碰到程序 bug 的时候，还可以使用 Bug Report 表单向 Konsole 开发人员提交问题。

相较于另一个流行的软件包 xterm，Konsole 只能算是“年轻一辈”了。下一节我们要探望一下“老古董” xterm。

## 6.使用xterm终端仿真器

最古老也是最基础的终端仿真器软件包是 xterm。xterm 在 X Window（历史上流行的显示服务器）出现之前就有了，如今仍是一些发行版（比如 openSUSE）的默认配备。

xterm 是一个功能完善的仿真软件包，并不需要太多的资源（比如内存）来运行。正因为如此，在专门为老旧硬件设计的 Linux 发行版中，xterm 非常流行。

尽管 xterm 并没有提供很多炫目的特性，但是把一件事做到了极致：仿真旧式终端，比如数字设备公司（digital equipment corporation，DEC）的 VT102、VT220 以及 Tektronix 4014 终端。对于 VT102 和 VT220 终端，xterm 甚至能够仿真 VT 序列色彩控制码，允许你在脚本中使用色彩。

> **注意**
>
> DEC VT102 和 VT220 是盛行于 20 世纪 80 年代和 90 年代初期，用于连接 Unix 系统的哑文本终端。VT102/VT220 不仅能显示文本，还能使用块模式图形显示基本的图形结构。由于这种终端访问方式如今仍在很多商业环境中使用，因而使得VT102/VT220 仿真依然流行。

下图展示了运行在 CentOS 发行版的 GNOME Shell 环境中的 xterm（必须手动安装）。可以看出它非常“朴素”。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/018-21ebbd957c806df5099163a8c871747b-18aebd.jpg" alt="{%}" style="zoom:80%;" />

如今想要把 xterm 终端仿真器找出来可得花点儿心思。它通常并没有被包含在桌面环境的图形菜单中。

### 6.1 访问xterm

在 KDE 桌面环境（Plasma）中，可以通过点击屏幕左下角的 Application Launcher 图标，然后点击 Applications ➪ System ➪ standard terminal emulator for the X Window system （xterm）来访问 xterm。

只要安装了 xterm 软件包，就可以通过 GNOME Shell 的搜索功能访问 xterm。点击桌面左上角的 Activities 图标，出现搜索栏时，在其中输入 xterm，就会看到 Konsole 图标出现。另外别忘了，你可以自己创建快捷键启动 xterm。

xterm 包允许使用命令行参数设置各种特性。接下来将讨论这些特性以及如何进行修改。

### 6.2 命令行选项

xterm 的命令行选项非常多。你可以控制大量的特性来定制终端仿真，比如允许或禁止某种 VT 仿真。

> **注意**
>
> xterm 的配置选项数量众多，无法在此一一列举。bash 手册中提供了大量的参考文档。第3章中会讲到如何阅读 bash 手册。另外，xterm 开发团队也在其网站上提供了一些不错的帮助资料。

可以通过向`xterm`命令加入参数来调用某些配置选项。如果想让 xterm 仿真 DEC VT100 终端，可以输入命令`xterm -ti vt100`，然后按 Enter 键。下表给出了一些可以配合 xterm 终端仿真器使用的命令行选项。

**xterm命令行选项**

| 选项             | 描述                       |
| :--------------- | :------------------------- |
| `-bg *color*`    | 指定终端背景色             |
| `-fb *font*`     | 指定粗体文本所使用的字体   |
| `-fg *color*`    | 指定用于前景文本的颜色     |
| `-fn *font*`     | 指定文本字体               |
| `-fw *font*`     | 指定宽文本字体             |
| `-lf *filename*` | 指定用于屏幕日志的文件名   |
| `-ms *color*`    | 指定文本光标颜色           |
| `-*name*`        | 指定标题栏中的应用程序名称 |
| `-ti *terminal*` | 指定要仿真的终端类型       |

一些 xterm 命令行选项使用加号（`+`）或减号（`-`）来指明如何设置某种特性。加号表示启用某种特性，减号表示关闭某种特性，反之亦然。加号可以表示禁止某种特性，减号可以表示允许某种特性，比如在使用`bc`参数的时候。下表中列出了可以使用`+/-`命令行选项来设置的一些常用特性。

**xterm `+/-`命令行选项**

| 选项         | 描述                                        |
| :----------- | :------------------------------------------ |
| `ah`         | 启用 / 禁止文本光标高亮                     |
| `aw`         | 启用 / 禁止文本行自动环绕（auto-line-wrap） |
| `bc`         | 启用 / 禁止文本光标闪烁                     |
| `cm`         | 启用 / 禁止识别 ANSI 色彩更改控制码         |
| `fullscreen` | 启用 / 禁止全屏模式                         |
| `j`          | 启用 / 禁止跳跃式滚动（jump scrolling）     |
| `l`          | 启用 / 禁止将屏幕数据记录进日志文件         |
| `mb`         | 启用 / 禁止边缘响铃（margin bell）          |
| `rv`         | 启用 / 禁止图像反转                         |
| `t`          | 启用 / 禁止 Tektronix 模式                  |

注意，不是所有的 xterm 实现都支持这些命令行选项。在启动 xterm 时，可以使用`-help`来确定所使用的 xterm 实现支持哪些选项。

> **注意**
>
> 如果觉得 xterm 还不错，但是想使用更现代的终端仿真器，那么不妨考虑 rxvt-unicode 软件包。可以通过大多数发行版的标准仓库（参见第 9 章）来安装 rxvt-unicode，它占用内存不多，运行速度飞快。

现在你已经了解了 3 种终端仿真软件包，一个重要的问题是：谁才是最好的终端仿真器？对于这个问题，没有什么权威的答案。究竟使用哪个仿真器软件包取决于你的个人需求。不过能够有所选择总是件好事。

## 7.小结

开始学习 Linux 命令行命令前，需要先能访问 CLI。在图形化界面的世界里，有时这会费点儿周折。本章讨论了能够进入 Linux 命令行的各种界面。

首先，我们讨论了通过虚拟控制台终端（GUI 之外的终端）以及图像化终端仿真软件包（GUI 中的终端）访问 CLI 时的不同，简要对比了两种访问方式之间的差异。

接下来，我们详细探究了通过虚拟控制台终端访问 CLI，包括像更改背景色这类控制台终端配置选项。

在学习了虚拟控制台终端之后，我们还讲述了通过图形化终端仿真器来访问 CLI，其中主要涉及 3 种终端仿真器：GNOME Terminal、Konsole 和 xterm。

本章还介绍了 GNOME Shell 桌面项目的 GNOME Terminal 终端仿真软件包。GNOME 桌面环境通常已经默认安装了 GNOME Terminal，可以通过其菜单项和快捷键方便地设置很多终端特性。

除此之外，我们还讨论了 KDE 桌面项目的 Konsole 终端仿真软件包。KDE 桌面环境（Plasma）通常已默认安装了 Konsole。它提供了一些非常好的特性，比如能够监测到空闲的终端。

本章最后讲到的是 xterm 终端仿真器软件包。xterm 是 Linux 中第一个可用的终端仿真器，能够仿真旧式终端硬件，比如 VT 和Tektronix 终端。