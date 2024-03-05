# Ubuntu安装JDK

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122000356.png)

## 下载JDK
* 官网下载
* 使用wget命令下载
```
# jdk-8u141-linux-x64.tar.gz
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz" tar xzf jdk-8u141-linux-x64.tar.gz
```
* 华为云 : https://repo.huaweicloud.com/java/jdk/

## 解压下载的tar.gz文件，指定解压路径

```
~/Softwares/java/
``` 

## 创建目录 (xxx 换成你自己系统账户名称)
```
sudo mkdir /home/xxx/Softwares/java/
```
    
## 执行解压命令
```
sudo tar -zxvf jdk-7u60-linux-x64.tar.gz -C /home/xxx/Softwares/java/
```

## 将解压得到的文件夹改名为: jdk   (假设原文件夹名称为 www)
```
cd /home/xxx/Softwares/java/
mv www jdk
```

## 设置环境变量

* 在（ ~/.bashrc ）文本末尾加上：

```
## 或者这个文件 /etc/profile
vim ~/.bashrc
```

```
#set oracle jdk environment
export JAVA_HOME=home/xxx/Softwares/java/jdk
export JRE_HOME=${JAVA_HOME}/jre  
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
export PATH=${JAVA_HOME}/bin:$PATH  
```
    
## 使环境变量生效
```
source ~/.bashrc
```
    
## 验证
```
java -version
```

```
javac -version
```
