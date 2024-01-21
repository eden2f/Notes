# UEditor 图片在线管理问题修复

### 问题

图片能够正常上传，正常显示，但是在线管理那里的图片显示不了。

![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121210113.png)


## 解决方法
    
### 修改 jsp/controller.jsp 的代码
    
* 修改前

```
request.setCharacterEncoding( "utf-8" );
response.setHeader("Content-Type" , "text/html");

String rootPath = application.getRealPath( "/" );

out.write( new ActionEnter( request, rootPath ).exec() );
```

* 修改后
   
```
request.setCharacterEncoding( "utf-8" );
response.setHeader("Content-Type" , "text/html");

String rootPath = application.getRealPath( "/" );

String action = request.getParameter("action");  
String result = new ActionEnter( request, rootPath ).exec();  
if( action!=null &&   
    (action.equals("listfile") || action.equals("listimage") ) ){  
    rootPath = rootPath.replace("\\", "/");  
    result = result.replaceAll(rootPath, "/");  
}   
out.write( result );
```
    
### 修改 ueditor源码中 FileManager.java 的代码
    
* 修改前

```
private String getPath ( File file ) {
    String path = file.getAbsolutePath();
    return path.replace(this.rootPath, "" );
}
```

* 修改后

```
private String getPath ( File file ) {
    String path = PathFormat.format(file.getAbsolutePath());
    return path.replace(this.rootPath, "" );
}    
```