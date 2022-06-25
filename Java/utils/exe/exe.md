## **准备**

准备工作：

- 一个jar包，没有bug能正常启动的jar包
- exe4j，一个将jar转换成exe的工具
- inno setup，一个将依赖和exe一起打成一个安装程序的工具

## **开始**

以我为例子，我将jar包放在了桌面

打开安装好的exe4j

<img src="imd/640" alt="图片"  />

直接下一步进入界面，选择JAVA转EXE

<img src="imd/641" alt="图片"  />

然后点下一步，输入名称和输出路径

<img src="imd/642" alt="图片"  />

继续点击下一步，选择启动模式

<img src="imd/643" alt="图片"  />

下方有个选项，需要设置打包后的程序兼容32和64位系统

<img src="imd/644" alt="图片" style="zoom:80%;" />

进来后勾选上

<img src="imd/645" alt="图片" style="zoom:80%;" />

然后一直下一步，一直出现如下界面，开始选择jar包以及配置

在VM参数配置的地方加上：-Dfile.encoding=utf-8

<img src="imd/646" alt="图片" style="zoom:80%;" />

<img src="imd/647" alt="图片" style="zoom:80%;" />

<img src="imd/648" alt="图片" style="zoom:80%;" />

<img src="imd/649" alt="图片" style="zoom:80%;" />

点击下一步，配置JRE

<img src="imd/650" alt="图片" style="zoom:80%;" />

下拉框点击后进入如下界面

<img src="imd/651" alt="图片" style="zoom:80%;" />

<img src="imd/652" alt="图片" style="zoom:80%;" />

照着这个样子写的目的是，最终会把本地jre目录和exe一起打包，让exe文件自己去根据路径去查找一起打包的jre，可不用再安装jdk

<img src="imd/653" alt="图片" style="zoom:80%;" />

接着下一步，选择Client VM

<img src="imd/654" alt="图片" style="zoom:80%;" />

然后一直下一步，最终出现如下界面

<img src="imd/655" alt="图片" style="zoom:80%;" />

这个时候你会发现桌面多了一个demo.exe文件，这个时候先别着急点开，接下来就是将jre和exe文件再打个包合并，达到在没有jdk电脑环境下也能运行。

打开inno setup，左上角File - New

<img src="imd/656" alt="图片" style="zoom:80%;" />

直接点下一步，填写配置，应用名称，版本等，随意

<img src="imd/657" alt="图片" style="zoom:80%;" />

然后点击下一步，这个地方默认就行，直接下一步

<img src="imd/658" alt="图片" style="zoom:80%;" />

接着选择生成好的exe文件

<img src="imd/659" alt="图片" style="zoom:80%;" />

然后下一步，进入这个界面保持默认，直接下一步

<img src="imd/660" alt="图片" style="zoom:80%;" />

依旧下一步，不用管

<img src="imd/661" alt="图片" style="zoom:80%;" />

继续下一步，这里是选择语言

<img src="imd/662" alt="图片" style="zoom:80%;" />

然后就是选择输出路径和填写安装程序的名字了

<img src="imd/663" alt="图片" style="zoom:80%;" />

然后下一步，直接点Next，然后结束

配置到最后一步了，脚本文件，到这里会弹出问你是否马上编译，选择否，先把脚本写好再自己编译：

<img src="imd/664、" alt="图片"  />

然后到了最后一步了，把本地的JRE写进脚本

<img src="imd/665" alt="图片" style="zoom:80%;" />

<img src="imd/666" alt="图片"  />

<img src="imd/639" alt="图片"  />

Source: "自己本地JRE路径*"; DestDir: "{app}{#MyJreName}"; Flags: ignoreversion recursesubdirs createallsubdirs

然后直接编译就好了，会提示保存当前脚本，随便起个名字，下个还可以继续用

<img src="imd/667" alt="图片" style="zoom:80%;" />

<img src="imd/668" alt="图片" style="zoom:80%;" />

然后等待绿色滚动条结束

<img src="imd/669" alt="图片" style="zoom:80%;" />

当绿色滚动条结束后，桌面会多了一个setup.exe文件

<img src="imd/670" alt="图片" style="zoom:80%;" />

也同时会跳出一个安装的，因为程序帮你自动启动生成的安装程序了，安装就可以了，安装的时候记得勾选创建快捷方式

<img src="imd/671" alt="图片" style="zoom:80%;" />

这个就是最后的程序了，双击运行就可以看到结果了，把setup.exe文件给别人安装，就都可以看到自己的程序了！