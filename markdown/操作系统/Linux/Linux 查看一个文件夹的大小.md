# Linux 查看一个文件夹的大小

> linux怎么查看一个文件夹的大小 : [https://blog.csdn.net/weixin_34148340/article/details/85943035](https://blog.csdn.net/weixin_34148340/article/details/85943035)

linux查看一个文件夹的大小的命令为：
```shell
du --max-depth 1 -lh  该文件夹的完整路径
```
例，查询/var文件夹的大小：
```shell
du --max-depth 1 -lh  /var
```
du 递归查询该路径下所有文件的大小（若不加任何参数，则显示文件夹内的所有文件，包括文件夹内子文件夹的内容）。
命令解释：
参数 --max-depth 1 -lh 设置递归深度为1，及不查询子文件夹。因而使用此参数只显示该文件夹的大小，不显示其中子文件夹的大小。
注意：
视操作系统版本不同，命令可能为：
```shell
 du --max-depth 1 -lh  该文件夹的完整路径
```
或：
```shell
du --max-depth=1 -lh  该文件夹的完整路径
```
