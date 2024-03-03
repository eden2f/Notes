# PicGo集成Gitee搭建图床

> [参考文章 -> https://www.jianshu.com/p/b69950a49ae2](https://www.jianshu.com/p/b69950a49ae2)

## 前言
本文图床是基于『PicGo + Gitee』实现的
常用的图床实现方式，有如下几种

1. 利用GitHub、Gitee等免费的代码托管网站，进行图片的存储及访问 
   1. 优点：免费
   2. 缺点：GitHub响应慢，用Gitee没有该问题
2. 利用各类博客网站，将图片上传保存在文档中 
   1. 优点：免费
   2. 缺点：网站文档丢失、自己误删、后续想迁移不方便
3. 使用各种云服务，如腾讯云COS、阿里云OSS、七牛图床等 
   1. 缺点： 要钱
## 实现
### 安装

- PicGo 软件
- gitee 2.0.2 插件
#### PicGo安装

1.  PicGo应用下载链接 -> [https://github.com/Molunerfinn/PicGo/releases](https://github.com/Molunerfinn/PicGo/releases)]([https://github.com/Molunerfinn/PicGo/releases](https://github.com/Molunerfinn/PicGo/releases)) 
2.  安装之后打开主界面 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003537.png#id=PUG5G&originHeight=450&originWidth=800&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### gitee插件安装
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003600.png#id=hdfYu&originHeight=450&originWidth=800&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 初始化 Gitee 仓库

1.  注册码云的方法很简单，网站引导都是中文，不多说了，我们直接建立自己的图床库。 
2.  点击右上角的+号，新建仓库 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003621.png#id=t1mfO&originHeight=230&originWidth=313&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

3. 新建仓库的要点如下： 
   1. 输入一个仓库名称
   2. 其次将仓库设为公开
   3. 勾选使用Readme文件初始化这个仓库 
      1. 这个选项勾上，这样码云会自动给你的仓库建立master分支，这点很重要!!!我因为这点折腾了很久，因为使用github做图床picgo好像会自动帮你生成master分支，而picgo里的gitee插件不会帮你自动生成分支。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003645.png#id=ck21m&originHeight=848&originWidth=730&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 配置PicGo
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003709.png#id=MTWAj&originHeight=450&originWidth=800&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

1. 配置插件的要点如下： 
   - owner：用户名
   - repo：仓库名称 
      - ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003729.png#id=CESYa&originHeight=75&originWidth=529&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
   - token：填入码云的私人令牌
   - path：路径，可以写imgs
   - message：每上传一张图片，都会产生一个Commit，message就是Commit的描述，可不填
#### 这个token怎么获取，下面登录进自己的码云

1.  点击头像，进入设置 
   1. ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003753.png#id=b0KeV&originHeight=227&originWidth=137&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
2.  找到右边安全设置里面的私人令牌 
   1. ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003815.png#id=DXKz4&originHeight=213&originWidth=226&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
3.  点击`生成新令牌`，把**projects**这一项勾上，其他的不用勾，然后提交 
   1. ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003837.png#id=uuvRz&originHeight=448&originWidth=530&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
   2. 这里需要验证一下密码，验证密码之后会出来一串数字，这一串数字就是你的token，将这串数字复制到刚才的配置里面去。 
      1. 注意：这个令牌只会明文显示一次，建议在配置插件的时候再来生成令牌，直接复制进去，搞丢了又要重新生成一个。
      2. ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304003900.png#id=CS7MF&originHeight=309&originWidth=458&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
4.  现在保存你刚才的配置，然后将它设置为默认图床，大功告成。 
