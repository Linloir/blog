---
title: 用 adb + app_process 执行 Java 代码 —— 一种无需安装 apk 的脱机代码执行方案
date: 2024-10-20 21:22:40
tags:
  - 安卓
  - 综合
  - 工作
categories:
  - 技术
---

## 方案速览

本方案本质上是使用了安卓提供的 `app_process` 命令，在将 Java 代码正确地打包为需要的 `.jar` 或是 `.dex` 文件后，通过 `app_process` 启动对应的入口函数来实现 `adb` 执行 Java 代码的能力。

对于目标 `.jar` 或是 `.dex` 文件，有两种不同的编译方案：

1. `.dex` 文件方式：
   1. 创建一个普通的 Java 工程，正常地添加依赖，**这里需要正确配置确保依赖在构建的时候也会被打包进制品 jar 中**，编写相关代码
   2. 构建 jar 包
   3. 在安卓 SDK 文件夹下 (一般为 `<homedir>\AppData\Local\Android\Sdk`)，找到 `cmdline-tools\latest\bin\d8.bat` (这里可以将 `<homedir>\AppData\Local\Android\Sdkcmdline-tools\latest\bin` 添加到环境变量中方便后面调用 `d8` 命令)，如果没有，可以在 Android Studio 更新 Commandline Tools 或是在 `build-tools\<version>` 下找到一个能用的
   4. 使用 `d8 [options] <source jar>` 命令编译出 `.dex` 文件，例如 `d8 server.jar`，会在当前工作目录下生成 `classes.dex`
2. `.jar` 文件方式：
   1. 创建一个 Empty Activity 的 Android Studio 安卓工程，正常编写 Java 代码 (在 `app/src/main/java` 下) 以及添加依赖
   2. 使用 Android Studio 的构建 apk 能力打包一个 apk (例如 `app-debug.apk`)
   3. 将 `.apk` 后缀改为 `.jar`

得到 `.jar` 或是 `.dex` 文件以后，使用 `adb [-s serial] push <source file> <target>` 命令推送到手机端

最后，在 `shell` 环境执行 `app_process` call 起 Java 程序：

`app_process` 启动命令为 `app_process -Djava.class.path=<path to jar> [other java options] <working directory (preferred /)> com.your.package.Class [args] ...`，也即对于打包好的 `server.jar` 文件，执行命令可能为 `adb shell app_process -Djava.class.path=/data/local/tmp/server.jar / com.linloir.server.Main --port 23456`

---

## 背景

起因是想要做一个检查安卓设备网络连通性的能力，最初想的就是 `adb ping` 直接秒了，甚至还可以复杂点用 `adb ip route get 1.1.1.1` 拿到网关以后分别去 ping 网关和外网地址，从而判断设备到网关的连通性和网关到外网的连通性

但是很快遇到了一个很诡异的现象：某些设备连续 ping 同一个地址 (即 A 先 ping 网关，然后 B ping 网关，最后 C ping 网关，串行执行) 时，会有一些设备出现 80% 以上的丢包甚至直接完全无法 ping 通的情况，并且当某台设备 ping 不通的同时，再开一个 shell 让它 ping 另外一个地址 (例如外网地址)，又是完全正常的，并且同一时间之前卡住的还是完全卡住的状态。反复研究发现有那么一些手机互相就像有羁绊一样，只要挨个 ping 相同地址就基本后一台铁挂，由于时间有限同时能力也有限，暂时没有深究了 (如果有后续就再补一篇文章)

于是，就因为这个 ping 这样谎报军情的现象，导致最后实在没办法再采用原本的方案，被迫不得不去找一个新的、能够兼容所有主流安卓设备的方案

思路很快确定了，就是做一个 HTTP 服务，给个 /ping 接口，客户端这边去发 GET 请求然后确认回包 200 即可。但是，问题就在于，如何能够在所有主流安卓设备上发 HTTP 请求然后判断结果呢？

最初，我想到了 `curl`，然后发现不是所有的设备都有 `curl`；于是转而投奔 `nc`，结果发现 `nc` 也不是所有的设备都有；其他的命令其实基本就也不想再试了，因为就算当前手头上的设备都能支持，谁也说不好会不会新来的机器就不支持了...

这时候都感觉仿佛只能用最最丑陋的下策了 —— 装一个 apk 包然后用 adb 拉起来然后走 socket 通信去执行 HTTP 请求。但是这样的方案，在自动化工程里面几乎是不可接受的，比如，光是自动化安装包这个环节就有各种坑可以踩，就算手动装也有不小的工作量；同时，call 起 apk 的过程也可能影响前台的 app。总之，这个方案一定是利大于弊的，因此也就没有第一时间去做尝试，而是继续去寻找可行的方案

经过同事点拨，突然想到，平常自动化测试用到的 `openatx/uiautomator2` 这个包，它本质上就是个 python 包装的反向代理，使得电脑端通过 python API 和手机侧的 server 通信，再把请求转换成 uiautomator 的指令加以执行。那么既然 `openatx/uiautomator2` 是这样的 server-client 架构，那它肯定在手机上也跑了个 server 服务吧，这个服务之前有注意到就是一个 `u2.jar` 文件，所以应该意味着，我应该也可以写一个 Java 程序然后想办法在安卓端跑起来

然后就开始了后文中试图让 Java 能在 adb shell 中跑起来的各种尝试...

## 尝试

### 通过 dalvikvm 执行 .dex 文件

最初的思路比较直接，安卓可以说是基于 Java 的，早期的运行时虚拟机 dalvikvm 即便到了 ART 时代仍然被系统所兼容，并且 `dalvikvm` 命令也仍然存在于所有安卓设备上，于是便试图通过 `dalvikvm -cp <dex file> com.your.package.Class` 来执行代码

在 demo 验证阶段，一个简单的 Hello World 程序是可以运行的，并且所有的设备都能够正确的执行，这基本证明了两点：

1. 使用 Java 来编写兼容大部分主流安卓设备的特殊逻辑代码并在设备上执行这一思路是可行的
2. dalvikvm 确实可以执行 Java 程序

但是很快，在 HTTP 请求的代码验证中，部分设备就出现了各种各样的错误：

1. 触发 Segmentation fault
2. 触发异常 `java.lang.UnsatisfiedLinkError` 并且直接无法执行
3. 触发 `java.lang.ClassNotFoundException` 异常但是不影响执行

其中，第二种异常提供了一些有用的信息：

```text
java.lang.reflect.InvocationTargetException
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.okhttp.OkUrlFactory.startBoost(OkUrlFactory.java:127)
        at com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:79)
        at com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:69)
        at com.android.okhttp.HttpHandler.openConnection(HttpHandler.java:44)
        at java.net.URL.openConnection(URL.java:993)
        at HelloWorld.sendGetRequest(HelloWorld.java:64)
        at HelloWorld.main(HelloWorld.java:26)
Caused by: java.lang.UnsatisfiedLinkError: No implementation found for android.os.IBinder com.android.internal.os.BinderInternal.getContextObject() (tried Java_com_android_internal_os_BinderInternal_getContextObject and Java_com_android_internal_os_BinderInternal_getContextObject__)
        at com.android.internal.os.BinderInternal.getContextObject(Native Method)
        at android.os.ServiceManager.getIServiceManager(ServiceManager.java:40)
        at android.os.ServiceManager.getService(ServiceManager.java:56)
        at com.oppo.luckymoney.LMManager.init(LMManager.java:209)
        at com.oppo.luckymoney.LMManager.<init>(LMManager.java:197)
        at com.oppo.luckymoney.LMManager.getLMManager(LMManager.java:202)
        ... 8 more
```

它指出，即便我在上层使用的是 `java.net.URL.openConnection`，下层应用的实现还是变成了 `com.android.okhttp.OkUrlFactory` 这种设备相关的代码，这应该就是导致在不同设备上会有不通执行结果的原因

进一步猜测，`UnsatisfiedLinkError` 指出部分动态链接库没有被正确地加载，考虑到对于一个普通程序而言，在启动时安卓运行时肯定会进行一些初始化操作，其中肯定有一部分链接库会在这个阶段由运行时去完成链接，而 `dalvikvm` 作为直接与虚拟机交互的指令，很有可能就是缺少了这样一些准备操作导致一些相关的运行时库没有被正确链接上，进而导致了 `UnsatisfiedLinkError` 甚至是 Segmentation Fault

依据这样的猜测，dalvikvm 应当是行不通了，势必要找到一个与安卓运行时挂钩的东西去执行 `.dex` 才能够避免同样的问题

### 通过 uiautomator 1.0 执行 .jar 文件

目光再次转向 `atx/uiautomator` 工程，在 readme 中读到了 uiautomator 1.0 时代能够通过 `uiautomator <jar> ... [-c class]` 的方式来执行 `.jar` 文件中自动化测试的代码，遂猜测其也是使用这样的方式启动的 `u2.jar` 文件。(其实这里犯了一个很大的错误让我与真相擦肩而过，进而浪费了许多时间在这一步上，这一点将在 [经验与教训](#经验与教训) 一节详述)

网上能找到的文档大致描绘了一个这样的步骤：

1. 创建一个 Java 项目
2. 添加 `android.jar` 和 `uiautomator.jar` 依赖
3. 使用 `android create uitest-project -n <name> -t <target version that match android.jar and uiautomator.jar> -p <path to project>` 创建 xml
4. 使用 `ant` 构建 jar 包
5. 使用 `uiautomator <jar> ... [-c class]` 命令启动

很遗憾的是，当前 Android SDK 中的 `android` 命令早已不包含 `create uitest-project` 这一条指令了，并且官方早就删除了与 uiautomator 1.0 相关的文档。同时，现有的各种打包文档也都含糊不清，IDE 也还是 Eclipse，想要在当前的环境成功打包出来可以说是难上加难了。我尝试下载了旧版的 SDK、ant 以及 Java JDK8 来尝试完成这个工作，总是在各种环节出现不兼容的问题，遂最终也作罢。事后复盘想想估计就算真的成功编译了，估计是否兼容如今的安卓版本也是个大问题，并且要是别人想要复现一次这个过程也要遭受这样的苦难，实在不是一个值得尝试的思路

## 最终方案

最终，是在一篇关于 [scrcpy 源码解读](https://codezjx.com/posts/scrcpy-source-code-analysis/) 的博客中找到了一段关键的内容:

> 上面我们提到 `scrcpy-server.jar` 是通过 `app_process` 运行起来的，那么 `app_process` 又是什么鬼？`app_process` 是启动 zygote 和其他 Java 程序的应用程序，它可以让虚拟机从 `main()` 方法开始执行一个 Java 程序。具体的用法可以参考 `app_process` 源码中的注释：
>
> ```text
> "Usage: app_process [java-options] cmd-dir start-class-name [options]\n"
> ```
>
> 与 APP 进程不同，通过 `app_process` 启动的进程可以在 root 权限和 shell 权限 (adb 默认) 下启动，也就分别拥有了调用不同 API 的能力。通常情况下 shell 权限启动的 `app_process` 只能够调用一些能够完成 `adb` 本身工作的 API，root 权限启动的 app_process 进程则拥有更多权限，甚至能够调用系统 signature 保护级别的 API 及访问整个文件系统。
>
> 实际上不少 `adb` 命令都是对调用 `app_process` 进行了一些封装，这里举我们平时常用的 `am` 指令为例，我们在执行 `adb shell am` 的时候其实是执行了以下的脚本。通过 `app_process` 运行 `am.jar` 中 `com.android.commands.am.Am` 这个类的 `main()` 函数来完成具体操作。
>
> ```bash
> #!/system/bin/sh
> if [ "$1" != "instrument" ] ; then
>     cmd activity "$@"
> else
>     base=/system
>     export CLASSPATH=$base/framework/am.jar
>     exec app_process $base/bin com.android.commands.am.Am "$@"
> fi
> ```
>
> 这里需要注意的是 `app_process` 只能运行原始 dex 文件，也可以接收包含 `classes.dex` 的 `jar` 包，如 `am.jar`。这里 scrcpy 取巧直接用了编译 `apk` 的方式，再直接重命名为 `jar` 包，这样也可以被 `app_process` 运行。而且可以使用 gradle 方便的进行编译，直接在 Android Studio 中开发，调用原生 API，像源码中的 `IRotationWatcher.aidl` 也可以直接生成相应的 `IPC Proxy` 类进行调用，省力又省心。
>
> `scrcpy-server.jar` 在 `adb` 下通过 `app_process` 运行起来后，默认就有了 shell 权限，像一些截屏、录屏、模拟按键点击等功能都可以直接使用，而且完全不需要任何权限的声明。

也就是说，`app_process` 就是我最开始想要找的，带完整运行时版本的 `dalvikvm`

在引用的文章中，也提到了 scrcpy 使用的一种打包方法：

1. 创建一个 Empty Activity 的 Android Studio 安卓工程，正常编写 Java 代码 (在 `app/src/main/java` 下) 以及添加依赖
2. 使用 Android Studio 的构建 apk 能力打包一个 apk (例如 `app-debug.apk`)
3. 将 `.apk` 后缀改为 `.jar`

进一步分析，其实本质上就是借用了 Android Studio 将 Java 代码打包为 dex 文件的能力，真正使用的还是 apk 文件中的 `classes*.dex` 文件，通过以 `.zip` 打开构建的 apk 并删除其他无关文件再测试可以证实这一点。而由于 dex 本质上还是在给 Java 解释器提供 classpath，只是可能不同于 jar 文件的组织方式，因此理论上对于其他方式打包的 dex 文件，`app_process` 应该也是支持的

这里想到前面用到的 `d8.bat` 工具，其就是将 `.jar` 文件转换为 `.dex` 文件，遂进行测试，证实了另一种打包的思路：

1. 创建一个普通的 Java 工程，正常地添加依赖，**这里需要正确配置确保依赖在构建的时候也会被打包进制品 jar 中**，编写相关代码
2. 构建 jar 包
3. 在安卓 SDK 文件夹下 (一般为 `<homedir>\AppData\Local\Android\Sdk`)，找到 `cmdline-tools\latest\bin\d8.bat` (这里可以将 `<homedir>\AppData\Local\Android\Sdkcmdline-tools\latest\bin` 添加到环境变量中方便后面调用 `d8` 命令)，如果没有，可以在 Android Studio 更新 Commandline Tools 或是在 `build-tools\<version>` 下找到一个能用的
4. 使用 `d8 [options] <source jar>` 命令编译出 `.dex` 文件，例如 `d8 server.jar`，会在当前工作目录下生成 `classes.dex`

经过测试，其中第 4 点是必须的，原因在引用的文章中也有提到：*这里需要注意的是 `app_process` 只能运行原始 dex 文件，也可以接收包含 `classes.dex` 的 `jar` 包*，也就是说对于 `app_process`，`.jar` 可能只是被作为一个文件夹看待而不是作为实际的 classpath 看待，实际作为 classpath 的是提供的 `.dex` 或是 `.jar` 中的 `.dex`

由此可以看出，理论上只要能打包出带完整依赖的 `jar` 包，辅以 `d8` 的转换，就可以用来作为 `app_process` 的 classpath 了，只是使用 Android Studio 方案的明显优势就是对于 Android API 的天然支持和对于依赖管理和导出的便捷性吧，这里可以根据具体的需求和场景来选择

在有了 `.jar` 或是 `.dex` 后，参考 `app_process` 的启动参数：

```text
app_process [java-options] cmd-dir [--application | --zygote [--start-system-server]] start-class-name [options]

app_process executes a interpreted runtime environment for dalvik bytecodes in an application-alike environment

if together with --application option, System.out is redirected to logcat.
otherwise, System.out remains untouched, i.e., the output is the stdout.

app_process becomes zygote if starts with --zygote,

if together with --start-system-server option, zygote also starts system server by forking itself.
app_process can be used as a general program besides starting zygote.

java-options: Options of dalvik/libart
cmd-dir: Root path of the running process (e.g., root path of file operations)
start-class-name: Class to run
options: Options of the running class
```

即对于打包好的 `server.jar` 文件，执行命令可能为 `adb shell app_process -Djava.class.path=/data/local/tmp/server.jar / com.linloir.server.Main --port 23456`

## 经验与教训

1. 在实现需要部署在多设备上的需求时，优先考虑平台或设备无关的实现方案，例如跨平台语言或是设备统一的解决方案 (如 apk)
2. 验证不一定要在原先的工程里摘代码验证，写 bash 也可以，例如 curl 的验证应当直接写 bash 脚本去在每个 adb devices 获取到的设备上验证，而不是频繁地修改主体工程里与这一块逻辑相关的 go 代码再通过单测函数验证，这样明显会浪费更多时间，bash 脚本 GPT 很快就能实现然后进行快速验证
3. 在研究 `atx/uiautomator2` 的实现的时候，先入为主地就认为作者使用了 `uiautomator <jar>` 这种方式调用，后面注意力都放在了查找作者针对 `u2.jar` 的打包方式和相关代码上了，以至于在仓库里搜索 `u2.jar` 的时候竟然没有注意到在搜索结果中 `core.py` 赫然有着 `command = "CLASSPATH=/data/local/tmp/u2.jar app_process / com.wetest.uia2.Main"` 这一启动方式，与答案擦肩而过并且进一步在寻找 uiautomator 1.0 的打包方式上浪费了大半天的时间
4. 对于一个陌生的仓库，如果想要了解它的源码实现，不一定非要自己去读，除了进行仓库里的关键词搜索，也可以去查找别人的源码解析，走个捷径
