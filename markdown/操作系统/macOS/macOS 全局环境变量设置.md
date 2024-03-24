# macOS 全局环境变量设置

> 引用自Apple社区问答 : [https://discussionschinese.apple.com/thread/251633370](https://discussionschinese.apple.com/thread/251633370)

## 问题
每次打开终端都需要source .bash_profile才能使用自己定义的环境变量,如何设置每次打开新的终端都生效
## 答案
首先，检查你的默认shell是什么
```shell
echo $SHELL
```
老版本的macOS的默认shell是/bin/bash，而新版本的macOS Catalina开始，默认shell改为zsh。
对于zsh，需要在.zshrc文件中进行配置。
再检查终端中的偏好配置，是否在通用中设置了特殊的shell。
