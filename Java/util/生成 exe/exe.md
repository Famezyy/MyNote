## **准备**

准备工作：

- 一个jar包，没有bug能正常启动的jar包
- exe4j，一个将jar转换成exe的工具
- inno setup，一个将依赖和exe一起打成一个安装程序的工具

## **开始**

以我为例子，我将jar包放在了桌面

打开安装好的exe4j

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosCjWoC8P53wAmJDkffNqticsKIBiat9obGRQVRuGc7w6icicKHjMMBr5icaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

直接下一步进入界面，选择JAVA转EXE

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosCjWoC8P53wAmJDkffNqticsKIBiat9obGRQVRuGc7w6icicKHjMMBr5icaA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后点下一步，输入名称和输出路径

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosHNmxribibH4WxFalmGjCsDN1z2jKJbkbuu4GxPIRLlCmOMrN21rerndg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

继续点击下一步，选择启动模式

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosPicyYv5XCZOFl5VtsXaPeTqDKeXchnMGs8WMt52cciaiahYicvKXudBmPQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

下方有个选项，需要设置打包后的程序兼容32和64位系统

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosl65HsrVael25NLgS9FsU2gdicdqhwnuN2u6haa3Bs1tico2GMAfpcmbw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

进来后勾选上

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwoshv3pVnCOsBiaPmhtWfGF4cl8LibKYDBQl0gZicic4ZjbsJPtDATCbpdRibg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后一直下一步，一直出现如下界面，开始选择jar包以及配置

在VM参数配置的地方加上：-Dfile.encoding=utf-8

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzE8VDniaHvbNLoy7c4s0bm8TXtPe0o3gEG9X6xew5rD714FJIg62j6iaPA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzE3bch0mCz64nCNffB0gOLFV5D6lQ3U1yzzZXwAwf1UB9KSHNibVgZpHA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzEv6DT729RIyKyHS2L5qEKWy17dQnpDSbSGmm8vAXdSA6N5mVU8Sxefg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzENo6FrW0IibiaBrCv9x7MjUicxiaibT3pbs7yxCRVHHx4eWN3Cu5wubRckXA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

点击下一步，配置JRE

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzEmYpickBlJRVVg7gPONrkiaMic3l1XK4DYCic0DVWd8giay2KOC0F50Wldmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

下拉框点击后进入如下界面

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzEybp1ClQ9DricBpvrXodSImHEheXHPanSQ6X4VdOGUpMYryPUibO6SSuA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosHKaGFV7CG1vgTrXGwOwYzr3yibGBk4D6WbUUMFrdYCrmYCkdGxbEHzg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

照着这个样子写的目的是，最终会把本地jre目录和exe一起打包，让exe文件自己去根据路径去查找一起打包的jre，可不用再安装jdk

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwos1ibRQHNl30o2w1G1AMgcoN0YlAWvljiaWdYL579puYKzAb3HGU2SPDoQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接着下一步，选择Client VM

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosw7jvP1ACcsotG3sRjU2uGnVFNfgZpmqZKub9tlGYBAmOc9RQMxgiayA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后一直下一步，最终出现如下界面

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosbhfIkXzkWwl1Dvewf7JxjdKY5VwtYQnQxmyqnKdcMzichUdn0KEkRVQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个时候你会发现桌面多了一个demo.exe文件，这个时候先别着急点开，接下来就是将jre和exe文件再打个包合并，达到在没有jdk电脑环境下也能运行。

打开inno setup，左上角File - New

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzEwWxibq4pCmxeWoGCrEe3kSa8eZfwqj9yegw7MfqXBhd61jS8QYkdS3A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

直接点下一步，填写配置，应用名称，版本等，随意

![图片](https://mmbiz.qpic.cn/mmbiz_png/sTnayibHfVq6hZBwCeiaCcZicKdTIJQ9zzEWIFQxT38MP9W5UG3wmDicav0akrdFwfDSNLsjFOa0EgCic3iaibKVUoHTg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后点击下一步，这个地方默认就行，直接下一步

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwoslZdvZSIolfwK52brzFvibgRkcWHl5hhwWdeWcbPUwX3OaiaTiancoRaMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接着选择生成好的exe文件

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwoslRcrOuqZ4DzlyM5x6SMU1AUicf4OL6vP7CQHVDBpHeNMWXqax9aVqYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后下一步，进入这个界面保持默认，直接下一步

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosKVeVuGPdbQGyXckicxkCg4ecnicwpFZaOEpzA1seXFFc4ZkzTZHe67TA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

依旧下一步，不用管

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosykBmxpWgK4Z3zOpd2vV5d32dJmTIuuHHiaj8ELVA5PiaGLCwJl28DYLA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

继续下一步，这里是选择语言

![图片](https://mmbiz.qpic.cn/mmbiz_png/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosGSNksQqQt9DWAyhJ0saqEyYrZZkHMJ9LEsoyMGeC8hTqbCKuQjiaB7A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后就是选择输出路径和填写安装程序的名字了

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosVIDicCvzibb80H08uMicHlVwopzrfibaZciaGic1yibrn2ZgRRqbeuib8OKyeg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后下一步，直接点Next，然后结束

配置到最后一步了，脚本文件，到这里会弹出问你是否马上编译，选择否，先把脚本写好再自己编译：

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosiacX0N3iaWTjaNahz1Gjhh2yoyJxU5mWvuibzIic8886WrqBv6YZGEudgA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后到了最后一步了，把本地的JRE写进脚本

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosOuicAIoaMkcC8UPdsnCb1Mobm2HDLoLibd8eQ5qbaRDl2Iiag9ibYVtAvQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosRol3RXibh1TN65zRBsaCY7ssOzSdeTotJTxoBzn4UaTXglJVybib7Z2w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosRol3RXibh1TN65zRBsaCY7ssOzSdeTotJTxoBzn4UaTXglJVybib7Z2w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Source: "自己本地JRE路径*"; DestDir: "{app}{#MyJreName}"; Flags: ignoreversion recursesubdirs createallsubdirs

然后直接编译就好了，会提示保存当前脚本，随便起个名字，下个还可以继续用

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosu3mibdRNgWyL0ickFkkkEGiaOiaa9uulMy5icKXhNcfRO7qVMjfkFpwVmRg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosWRt6VzmUv15QR0s2fImS60BXqHVW58H7GsiaNLAn1FLzXBm41k0FXPQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后等待绿色滚动条结束

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwoskqTiaGzsuqOic0ibSWmUWJIRx3LS2wjYCG0fSzEL9q0kZiamq1CIb0bcpw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当绿色滚动条结束后，桌面会多了一个setup.exe文件

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosoY9WqvoTSLJPbG1MGT3Qial8SIVWEn9dwxicRb98GbKqdqDbb3XIzNvg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

也同时会跳出一个安装的，因为程序帮你自动启动生成的安装程序了，安装就可以了，安装的时候记得勾选创建快捷方式

![图片](https://mmbiz.qpic.cn/mmbiz/TNUwKhV0JpQzGBznBYXGYDfiaJt9lwwosWRt6VzmUv15QR0s2fImS60BXqHVW58H7GsiaNLAn1FLzXBm41k0FXPQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个就是最后的程序了，双击运行就可以看到结果了，把setup.exe文件给别人安装，就都可以看到自己的程序了！