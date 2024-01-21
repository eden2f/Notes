# Ubuntu安装Tomcat

## 下载tomcat ( 以apache-tomcat-8.5.9.tar为例 )

* 官网下载
* 使用wget命令下载

## 解压下载的tar.gz文件，指定解压路径

###  需先创建该目录 
```
sudo mkdir /home/xxx/Softwares/tomcat/
```
    
### 执行解压命令
```
sudo tar -zxvf apache-tomcat-8.5.9.tar -C  /home/xxx/Softwares/tomcat/
```
    
### 修改解压得到的文件夹名称为: tomcat_8.5.9 （假设原名称为: www ）
```
mv www tomcat_8.5.9
```

## 设置tomcat JAVA_HOME
    
* ### 进入解压得到的文件夹的bin目录下
  ```
  cd /home/xxx/Softwares/tomcat/tomcat_8.5.9/bin/
  ```
    
### 修改 catalina.sh 文件
```
sudo vim catalina.sh
```
    
### 添加 JAVA_HOME 设置 (修改成自己的 JAVA_HOME )
    
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122000917.png)

## 启动tomcat
```
cd /home/xxx/Softwares/tomcat/tomcat_8.5.9/bin/
sudo ./startup.sh/
```
    
## 验证

浏览器访问 : localhost:8080/