# Electerm 开源的SSH/SFTP客户端 配Gitee绝了

> 博客原文引用 [又一款开源免费的SSH/SFTP客户端Electerm](https://www.xiaoz.me/archives/15692)
Github项目地址 [https://github.com/electerm/electerm](https://github.com/electerm/electerm)
Electerm主页 [https://electerm.github.io/electerm/](https://electerm.github.io/electerm/)

Electerm是一款基于electron开发的SSH/SFTP客户端，同时支持Linux、MAC、Windows操作系统，免费开源。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230124.png#id=KfleX&originHeight=549&originWidth=987&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 功能特点

- 作为终端/文件管理器或ssh / sftp客户端（类似于xshell）
- 全局热键可切换窗口可见性（类似于guake，默认值为ctrl + 2）
- 多平台（Linux，Mac，Win）
- 🇺🇸 🇨🇳 🇧🇷 🇷🇺 🇪🇸 🇫🇷 🇹🇷 🇭🇰支持多国语言（electerm-locales，欢迎提供/修复问题）
- 双击直接编辑远程文件（小的）。
- 使用内置编辑器（小的）编辑本地文件。
- 使用公钥+密码进行身份验证。
- Zmodem（rz，sz）。
- 透明窗口（Mac，Win）。
- 终端背景图像。
- 全局/会话代理。
- 快速命令
- 将书签/主题/快速命令同步到github / gitee secret gist
- 快速输入
## 下载并安装Electerm
Electerm支持Linux、MAC、Windows操作系统，直接前往项目地址：[https://github.com/electerm/electerm/releases](https://github.com/electerm/electerm/releases) 根据操作系统下载最新版本。
如果是Windows系统，可下载作者编译好的压缩包，比如 `eleterm-1.11.6-win-x64.tar.gz`，解压后双击 `electerm.exe`即可使用。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230344.png#id=mFmTh&originHeight=354&originWidth=790&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 一些设置
设置中文语言：
Electerm默认是英文界面，可打开设置将语言设置为中文，大概在如下图的位置。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230413.png#id=Dmgiw&originHeight=650&originWidth=1046&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
设置字体：
Electerm默认使用的字体个人感觉不好看，可在设置中对字体进行修改，推荐 `Consolas`和微软雅黑字体。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230435.png#id=W3CCc&originHeight=223&originWidth=571&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
设置同步：
Electerm可以将书签（服务器连接信息）同步到Github或Gitee，这篇文章以Gitee为例来进行说明。

1. 首先在Gitee后台生成一个新令牌：https://gitee.com/profile/personal_access_tokens，根据提示进行生成，生成后令牌只会出现一次，注意保存。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230456.png#id=FpT4a&originHeight=486&originWidth=645&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

2. 打开Electerm的同步功能，切换到 `gitee`，填写刚刚获取的Gitee令牌，并勾选“加密”同步，否则服务器密码等信息是明文同步的（非常危险），接着保存并上传设置即可。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230516.png#id=tPYaj&originHeight=717&originWidth=1135&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果更换了电脑也是按照这样设置，然后点下载设置即可将书签同步到本地。
如果不想使用同步功能的也可以在书签设置中进行导出、导入。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230621.png#id=OYPNH&originHeight=334&originWidth=533&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 一些不错的功能
自从用了FinalShell的快捷命令后爱不释手，Electerm也支持快捷命令（快速命令），可将常用且复杂的Linux命令添加为快速命令，从而提高效率。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230653.png#id=uIRfn&originHeight=499&originWidth=801&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
不足的是Electerm快捷命令不支持分组，也不支持同步和导出，换了设备又得重新录入命令。如果需求比较强烈的，可以到作者Github进行反馈。
点右下角的Terminal info可查看服务器CPU、内存、网络等信息。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230837.png#id=Kf9x3&originHeight=359&originWidth=772&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
Electerm每次启动会默认新建一个Powershell窗口，个人并不习惯使用Powershell，如果您安装了WSL子系统，可以将默认 execWindows设置为 `System32/wsl.exe`
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230909.png#id=Yt0on&originHeight=275&originWidth=592&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这样您每次打开启动的将是WSL子系统，不再是Powershell
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320230932.png#id=lIwIl&originHeight=275&originWidth=592&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 总结
Electerm整体来说还是不错的，并且开源免费、多平台支持，自带中文界面。Electerm算不上非常成熟，使用中可能会遇到一些BUG，但完全满足日常的服务器管理和使用，而且作者更新非常勤快，遇到BUG也可以在Github进行反馈。
