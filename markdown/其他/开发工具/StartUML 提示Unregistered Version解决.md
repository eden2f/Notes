# StartUML 提示Unregistered Version解决

> 博客原文引用 [xhBruce](https://blog.csdn.net/qq_23452385/article/details/116096800)

**StartUML并不好用,只是刚好新电脑没UML软件,而常用的JUDE的安装包也没在,先StartUML应急.(如果有更好UML软件记得推下)**
## npm安装asar
```bash
npm install -g asar
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319231116.png#id=kn5yZ&originHeight=188&originWidth=2591&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 解压StartUML中app.asar文件
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319231138.png#id=TOyYi&originHeight=668&originWidth=1499&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 修改解压app资源文件中app\src\engine\license-manager.js
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319231152.png#id=fMbxM&originHeight=768&originWidth=1396&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 重新app打包，并替换原来的 app.asar
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319231220.png#id=zVqgY&originHeight=723&originWidth=1301&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
