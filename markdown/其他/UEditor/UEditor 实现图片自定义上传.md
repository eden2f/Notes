# UEditor 实现图片自定义上传
    
## 修改前端上传图片访问路径
    
* 官方说明：

![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205650.png)

* 新建 diyindex.html
    
![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205722.png)
   
* diyindex.html 代码
    
```
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
    "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<title>使用自己定义的上传类</title>
<meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
<script type="text/javascript" charset="utf-8" src="ueditor.config.js"></script>
<script type="text/javascript" charset="utf-8" src="ueditor.all.min.js"> </script>
<!--建议手动加在语言，避免在ie下有时因为加载语言失败导致编辑器加载失败-->
<!--这里加载的语言文件会覆盖你在配置项目里添加的语言类型，比如你在配置项目里配置的是英文，这里加载的中文，那最后就是中文-->
<script type="text/javascript" charset="utf-8" src="lang/zh-cn/zh-cn.js"></script>

<style type="text/css">
    div{
        width:100%;
    }
</style>
</head>
<body>
<div>
<h1>使用自己定义的上传类</h1>
<textarea id="editor" type="text/plain" style="width:1024px;height:500px;"></textarea>
</div>

<script type="text/javascript">
UE.Editor.prototype._bkGetActionUrl = UE.Editor.prototype.getActionUrl;
UE.Editor.prototype.getActionUrl = function(action) {
    if (action == 'uploadimage' ) {
        return 'http://localhost:8080/_ueditor/UploadImageServlet';
    } else {
        return this._bkGetActionUrl.call(this, action);
    }
}

var ue = UE.getEditor('editor');
</script>
</body>
</html>
```
    
* 注意
    
![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205758.png)

## 新建 UploadImageServlet 处理前端上传请求、保存图片、返回信息配置web.xml 中
        
![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205814.png)

### UploadImageServlet 代码
  
* 属性

![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205833.png)

* doPost 方法代码
        
```
protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    System.out.println("------- ueditor 图片上传  -------");
        
    FileItemStream fileStream = null;
    boolean isAjaxUpload = request.getHeader( "X_Requested_With" ) != null;
    
    if (!ServletFileUpload.isMultipartContent(request)) {
        sendData( response, false, AppInfo.NOT_MULTIPART_CONTENT , null);
        return ;
    }
    
    ServletFileUpload upload = new ServletFileUpload(
            new DiskFileItemFactory());
    
    if ( isAjaxUpload ) {
        upload.setHeaderEncoding( "UTF-8" );
    }
    
    try {
        FileItemIterator iterator = upload.getItemIterator(request);

        while (iterator.hasNext()) {
            fileStream = iterator.next();

            if (!fileStream.isFormField())
                break;
            fileStream = null;
        }
        
        if (fileStream == null) {
            sendData( response, false, AppInfo.NOTFOUND_UPLOAD_DATA , null);
            return ;
        }
        
        // 得到 得到 新路径格式+名称格式
        String savePath = this.savePath;
        // 得到 原文件名          
        String originFileName = fileStream.getName();
        // 得到 原文件名后缀
        String suffix = FileType.getSuffixByFilename(originFileName);
        
        // 将 originFileName 去掉原文件名后缀   
        originFileName = originFileName.substring(0,
                originFileName.length() - suffix.length());
        // 得到 新路径格式+名称格式+后缀    
        savePath = savePath + suffix;

        // 得到  文件上传大小限制，单位B
        int maxSize = this.maxSize;

        // 得到允许上传的图片格式数组
        String[] allowTypes = this.allowTypes;
        if (!validType(suffix, allowTypes) ) {
            sendData( response, false, AppInfo.NOT_ALLOW_FILE_TYPE , null);
            return ;
        }
        
        // 根据 新路径格式+名称格式+后缀 以及 原文件名 产生新的保存路径和名称
        savePath = PathFormat.parse(savePath, originFileName);
        
        // 得到 保存至图片服务器的 图片访问路径前缀
        String rootPath = request.getSession().getServletContext().getRealPath( "/" );
        rootPath = rootPath.substring( 0 , rootPath.length() );
        String physicalPath =  rootPath + savePath;

        // 将上传图片保存至 硬盘
        InputStream is = fileStream.openStream();
        State storageState = StorageManager.saveFileByInputStream(is, physicalPath , maxSize);
        is.close();
        
        if (storageState.isSuccess()) {
            storageState.putInfo("url", PathFormat.format(savePath));
            storageState.putInfo("type", suffix);
            storageState.putInfo("original", originFileName + suffix);
        }
        
        sendData( response, true, 0 , storageState);
        return ;
        
    } catch (FileUploadException e) {
        sendData( response, false, AppInfo.PARSE_REQUEST_ERROR , null);
        return ;
    } catch (IOException e) {
    }
    sendData( response, false, AppInfo.IO_ERROR , null);
    return ;
}
```
* 其他方法

![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205858.png)

* 注意
    
UploadImageServlet中的rootPath与config.json中的imageUrlPrefix需要按需配置！

```
// 图片服务器保存路径前缀
private String rootPath = "";
```

![](https://gitee.com/eden2f/pic-hosting/raw/master/notes/20240121205928.png)