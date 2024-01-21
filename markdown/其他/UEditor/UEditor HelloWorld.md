# UEditor HelloWorld

**注意：本文档仅适用于1.4.0之后的Java版UEditor。**
    
## 先决条件
1. JDK 1.6+
2. Apache Tomcat 6.0+
3. UEditor 1.4.0+
        
## 示例环境
* 软件版本信息
   * JDK 1.7
   * Tomcat 7
   * UEditor 1.4.3 UTF-8 Java版
   * OS: Windows10 
   * MyEclipse 

## 部署

1. 新建一个web工程或者maven工程 (这里以maven工程项目为例)

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121225033.png)

1. 从ueditor官网下载

[ueditor官网下载链接](http://ueditor.baidu.com/website/download.html)
    
3. 部署到新建的项目上
    
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121225139.png)
    
 * 将 jsp/lib 目录下的 jar 包复制到 WEB-INF/lib 目录下

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121225214.png)
 
 * 修改 jsp/config.json 的各个访问路径前缀，填为本项目在tomcat运行的访问前缀
 
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121225239.png)

## 运行项目
* 访问
```
http://localhost:8080/_ueditor/ueditor/index.html
```

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240121225304.png)
   
* 图片上传功能也可以正常使用了