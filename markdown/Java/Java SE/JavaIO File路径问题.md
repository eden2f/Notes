# JavaIO File路径问题

## 目录分隔符

在Windows中分隔符为'\'，在Unix/Linux中分隔符为'/'。

### 注意


在windows操作系统下,可以使用"\"与"/"作为目录分隔符,但是在Unix/Linux的操作系统下只能使用"/"作为目录分隔符。 

## 路径

### 绝对路径

指定文件的完整路径创建一个File对象，绝对路径一般以盘符开头。

### 相对路径

资源文件相对于对当前路径。
* .  代表是当前路径
* ..  代表是上一级路径

### 注意

如果当前路径与资源文件不是在同一个盘符下，没法写相对路径的。

## 示例

``` java
public class Demo2 {
	public static void main(String[] args) {
		/*
		File file = new File("F:"+File.separator+"a.txt");
		System.out.println("存在吗："+ file.exists());
		System.out.println("目录分隔符："+ File.separator);
		*/
		File file2 = new File("."); 
		System.out.println("当前路径："+file2.getAbsolutePath());
		File file = new File("..\\a.txt");
		System.out.println("存在吗："+ file.exists());
	}
}
```