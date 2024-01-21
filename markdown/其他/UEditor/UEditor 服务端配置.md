# UEditor 服务端配置

## 背景
UEditor 1.4.0 版本对之前的配置方式进行了简化，具体请参见：后端请求规范，为了适应这次升级，JAVA 后台也进行了重写，跟之前的版本差别较大，升级的用户注意阅读本文档。

本文档介绍 UEditor JAVA 后台的部署和配置说明。

注意：本文档仅适用于1.4.0之后的Java版UEditor。

## 先决条件
* JDK 1.6+
* Apache Tomcat 6.0+
* UEditor 1.4.0+
        
## 示例环境
* 软件版本信息
   * JDK 6u45
   * Tomcat 6.0.41
   * UEditor 1.4.3 UTF-8 Java版
   * OS: Windows7 Ultimate SP1 X64
   * Eclipse 4.4.0
* *软件路径信息

Tomcat 安装路径： 
```
D:\apache-tomcat-6.0.41\
```

## 部署

* 解压对应的UEditor压缩包至Tomcat的webapps目录下，最终，UEditor的安装路径为：

```
D:\apache-tomcat-6.0.41\webapps\ueditor1_4_3-utf8-jsp\
```

* 进入目录：

```
D:\apache-tomcat-6.0.41\webapps\ueditor1_4_3-utf8-jsp\ 
```

* 创建如下两个文件夹（注意区分大小写）：

```
WEB-INF\lib\
```

* 拷贝目录：

```
D:\apache-tomcat-6.0.41\webapps\ueditor1_4_3-utf8-jsp\jsp\lib\
```

目录下的所有jar包到第2步创建的lib目录下，结果如图所示：

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121223632.png)

* 部署完成，双击以下脚本文件，启动Tomcat（需要正确配置JAVA_HOME环境变量）。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121223657.png)

* 验证部署是否成功:
  * 在浏览器地址栏中输入如下URL：
   
   ```
   http://localhost:8080/ueditor1_4_3-utf8-jsp/jsp/controller.jsp?action=config
   ```
   
  * 出现类似下图所示内容，则配置成功，否则，即为失败。
   
    ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121224114.png)

* 通过 imageUrlPrefix 配置项给返回路径添加前缀

有一些情况下仅仅靠返回路径是不能得到正常的图片链接，需要通过配置 imageUrlPrefix 给插入图片的路径添加前缀。

有两种情况下需要配置 imageUrlPrefix：

应用程序目录不是网站根目录，需要给路径添加前缀。

在跨域上传的情况下，就需要配置imageUrlPrefix前缀。假设页面在 a.com 域下，文件上传到 b.com 域下，这样要配置 imageUrlPrefix 为 "http://b.com" 才能得到正常路径。

修改位置:

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121224146.png)

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121224214.png)

## 相关文档

* [详细配置 : http://fex.baidu.com/ueditor/#start-start](http://fex.baidu.com/ueditor/#start-start)