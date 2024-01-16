# 生成exe执行文件

## 1.外部工具

### **准备**

准备工作：

- 一个jar包，没有bug能正常启动的jar包
- exe4j，一个将jar转换成exe的工具
- inno setup，一个将依赖和exe一起打成一个安装程序的工具

### **开始**

以我为例子，我将jar包放在了桌面

打开安装好的exe4j

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/640-d25af754eefaad36a13f9e19315c5618-cb8892" alt="图片"  />

直接下一步进入界面，选择JAVA转EXE

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/641-d25af754eefaad36a13f9e19315c5618-d9c6cc" alt="图片"  />

然后点下一步，输入名称和输出路径

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/642-4ec4370031de9e81ca2a126324706a63-34178e" alt="图片"  />

继续点击下一步，选择启动模式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/643-7ece501dc0304b98fb375152ba2239c2-819f94" alt="图片"  />

下方有个选项，需要设置打包后的程序兼容32和64位系统

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/644-9ef1396b4f042009fe9723dd070480f1-2e1a45" alt="图片" style="zoom:80%;" />

进来后勾选上

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/645-9061ae324013c48e813853608cc58715-d75d59" alt="图片" style="zoom:80%;" />

然后一直下一步，一直出现如下界面，开始选择jar包以及配置

在VM参数配置的地方加上：-Dfile.encoding=utf-8

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/646-b50605d93463ffa8450844a9e5ca5ac8-31c8cd" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/647-2c7d85d36c0aeb484a71423b01194c42-e54b8f" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/648-eacb84525bcf714f9ce928eb45aef5f8-a8a49c" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/649-8a4e715abd0a7150f3bb7a7ea47ec5bb-4c3f32" alt="图片" style="zoom:80%;" />

点击下一步，配置JRE

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/650-58776fbeb5e228691c9d61d487dd09dc-7874de" alt="图片" style="zoom:80%;" />

下拉框点击后进入如下界面

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/651-bfeabb5d6a51201d33c933b4b0ce092c-8012e0" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/652-82629ee23127519a3b2815dbcea360b0-5bd1e5" alt="图片" style="zoom:80%;" />

照着这个样子写的目的是，最终会把本地jre目录和exe一起打包，让exe文件自己去根据路径去查找一起打包的jre，可不用再安装jdk

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/653-4cb7c19e3472a2583c36dbf182883914-abf56d" alt="图片" style="zoom:80%;" />

接着下一步，选择Client VM

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/654-fc8bc8452a43e3d559595c4c8d3488d0-5de481" alt="图片" style="zoom:80%;" />

然后一直下一步，最终出现如下界面

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/655-44f5911026033ec042b969640cd6b09e-08cd0c" alt="图片" style="zoom:80%;" />

这个时候你会发现桌面多了一个demo.exe文件，这个时候先别着急点开，接下来就是将jre和exe文件再打个包合并，达到在没有jdk电脑环境下也能运行。

打开inno setup，左上角File - New

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/656-8760d14760ad167e833a93d4e8bd6db1-59ba50" alt="图片" style="zoom:80%;" />

直接点下一步，填写配置，应用名称，版本等，随意

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/657-d8f74be4ef967796bd04fdbf5947de3c-be5a66" alt="图片" style="zoom:80%;" />

然后点击下一步，这个地方默认就行，直接下一步

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/658-13410c4adf35d17aef3f0d4db1bb7d44-215f90" alt="图片" style="zoom:80%;" />

接着选择生成好的exe文件

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/659-44bcfaed5979533b4c93247d2a8663dc-ae989d" alt="图片" style="zoom:80%;" />

然后下一步，进入这个界面保持默认，直接下一步

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/660-75a2730b486f3454c73370280bb58225-5136f5" alt="图片" style="zoom:80%;" />

依旧下一步，不用管

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/661-a3773c11cf472bb5bdd22764f5364bc7-6a89d5" alt="图片" style="zoom:80%;" />

继续下一步，这里是选择语言

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/662-05a176cb85db294ad97df58e23851b45-265a2d" alt="图片" style="zoom:80%;" />

然后就是选择输出路径和填写安装程序的名字了

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/663-cd9ce00dfa39c5af90073b3462eff016-0739e1" alt="图片" style="zoom:80%;" />

然后下一步，直接点Next，然后结束

配置到最后一步了，脚本文件，到这里会弹出问你是否马上编译，选择否，先把脚本写好再自己编译：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/664%E3%80%81-475cfb86c503785a95c373550a02dd3e-9d38f8" alt="图片"  />

然后到了最后一步了，把本地的JRE写进脚本

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/665-7dbf438d4f3f8ad8bc0654479a65960a-b3f056" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/666-7ee1dadc00e2b4ca8a0e9301442bd520-816571" alt="图片"  />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/639-7ee1dadc00e2b4ca8a0e9301442bd520-da4582" alt="图片"  />

Source: "自己本地JRE路径*"; DestDir: "{app}{#MyJreName}"; Flags: ignoreversion recursesubdirs createallsubdirs

然后直接编译就好了，会提示保存当前脚本，随便起个名字，下个还可以继续用

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/667-d7cfa4ff938684adf24a9d78f6785b0a-6152f9" alt="图片" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/668-ad76061941435c4ef9a8182f3bfb79fa-6e0848" alt="图片" style="zoom:80%;" />

然后等待绿色滚动条结束

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/669-925d94a37f5c4d7304da921880bf5c36-504a57" alt="图片" style="zoom:80%;" />

当绿色滚动条结束后，桌面会多了一个setup.exe文件

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/670-d3defcf7cecb07112e45863318806fd6-6fa2ce" alt="图片" style="zoom:80%;" />

也同时会跳出一个安装的，因为程序帮你自动启动生成的安装程序了，安装就可以了，安装的时候记得勾选创建快捷方式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/671-ad76061941435c4ef9a8182f3bfb79fa-d88e3f" alt="图片" style="zoom:80%;" />

这个就是最后的程序了，双击运行就可以看到结果了，把setup.exe文件给别人安装，就都可以看到自己的程序了！

## 2.Java Packager

JDK 16 引入的用于生成可执行文件的命令。

```bash
jpackage
	-- name installer
	-- app-version 1.0
	-- win-dir-shooser -win-console -win-shortcur
	--module-path modules/app.jar
	-- module org.myapp/org.mycompany.com.myapp.MainClass
```

## 3.使用native image
