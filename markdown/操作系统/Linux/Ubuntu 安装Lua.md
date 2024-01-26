# Ubuntu 安装Lua

## 下载

* [Lua-5.3.4 : http://www.lua.org/ftp/lua-5.3.4.tar.gz](http://www.lua.org/ftp/lua-5.3.4.tar.gz)
  * 浏览器等去官网下载
  * 使用wget命令下载

## 解压下载的tar.gz文件，指定解压路径

``` shell
/usr/local/lua 
```

* 需先创建该目录 
```
sudo mkdir/usr/local/lua
```

* 执行解压命令
```
sudo tar -zxvf lua-5.3.4.tar.gz -C mkdir /usr/local/lua/
```

* 将解压得到的文件夹改名为: lua-5.3.4   (假设原文件夹名称为 www)
```
cd /usr/local/lua/
mv www lua-5.3.4
```

## 进入该目录进行编译安装
```
cd /usr/local/lua/lua-5.3.4
make linux test 
make install
```
* 出现这个错误的话 : sudo apt-get install libreadline6 libreadline6-dev

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126230940.png)

## 创建软连接
```
ln -s /usr/local/lua/lua-5.3.4/lua  /usr/bin/lua
```
## 删除软连接
```
rm -rf /usr/bin/lua
```
    
## 测试

* 运行结果

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126231009.png)