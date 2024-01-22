## JavaIO File类常用方法

## 创建、重命名、移动

``` java
/*
通过File类常用的方法我们就可以获取以及修改文件 的属性数据。
	创建：
	createNewFile()	在指定位置创建一个空文件，成功就返回true，如果已存在就不创建然后返回false
	mkdir()			在指定位置创建目录，这只会创建最后一级目录，如果上级目录不存在就抛异常。
	mkdirs()		在指定位置创建目录，这会创建路径中所有不存在的目录。
	renameTo(File dest)	重命名文件或文件夹，也可以操作非空的文件夹，文件不同时相当于文件的剪切,剪切时候不能操作非空的文件夹。移动/重命名成功则返回true，失败则返回false。
 */
public class Demo3 {
	
	public static void main(String[] args) throws IOException {
		/*File file = new File("F:\\a.txt");
		File dir = new File("F:\\aa\\bb");
		System.out.println("创建一个空文件："+file.createNewFile());
		System.out.println("创建一个单级文件夹："+ dir.mkdir());
		System.out.println("创建一个多级文件夹："+ dir.mkdirs());
		
		操作文件：如果源文件与目标文件在同一级路径下，那么renameTo方法的作用是重命名，  如果源文件与目标文件不在同一级目录下，那么renameTo的作用就是剪切。 
		
		操作文件夹：如果源文件夹与目标文件夹在同一级路径下，那么renameTo方法的作用是重命名, 如果源文件夹与目标文件夹不在同一级目录下,那么renameTo不起作用（不能用于剪切文件夹）。 
		*/
		File file = new File("f:\\aa");
		File destFile = new File("E:\\bb");
		file.renameTo(destFile); 
	}
}
```

## 删除

``` java
/*
删除：
	delete()		删除文件或一个空文件夹，如果是文件夹且不为空，则不能删除，成功返回true，失败返回false。
	deleteOnExit()	在虚拟机终止时，请求删除此抽象路径名表示的文件或目录，保证程序异常时创建的临时文件也可以被删除

酷狗  在线试听... 

*/
public class Demo4 {
	
	public static void main(String[] args) {
		File file = new File("F:\\a.txt");
		System.out.println("删除成功吗？"+ file.delete());   //马上删除
//		file.deleteOnExit();  // deleteOnExit() 当jvm退出的时候执行删除动作。
		System.out.println("哈哈...");
		
	}
	
}
```

## 判断

``` java
/*
判断：
	exists()		文件或文件夹是否存在。
	isFile()		是否是一个文件，如果不存在，则始终为false。
	isDirectory()	是否是一个目录，如果不存在，则始终为false。
	
	isHidden()		是否是一个隐藏的文件或是否是隐藏的目录。
	isAbsolute()	测试此抽象路径名是否为绝对路径名。
*/
public class Demo5 {
	public static void main(String[] args) {
		File file = new File("..\\..\\a.txt");
		System.out.println("存在吗："+ file.exists());
		System.out.println("判断是否是一个文件："+ file.isFile());
		System.out.println("判断是否是一个文件夹："+ file.isDirectory());
		System.out.println("判断是否是一个隐藏文件："+ file.isHidden());
		System.out.println("是绝对路径吗？"+ file.isAbsolute());
	}
}
```

## 获取、查询

``` java
/*
获取：
	getName()		获取文件或文件夹的名称，不包含上级路径。
	getPath()       返回绝对路径，可以是相对路径，但是目录要指定
	getAbsolutePath()	获取文件的绝对路径，与文件是否存在没关系
	length()		获取文件的大小（字节数），如果文件不存在则返回0L，如果是文件夹也返回0L。
	getParent()		返回此抽象路径名父目录的路径名字符串；如果此路径名没有指定父目录，则返回null。
	lastModified()	获取最后一次被修改的时间。
*/
public class Demo6 {
	public static void main(String[] args) {
		File file = new File("f:\\a.txt");
		System.out.println("文件名："+ file.getName());
		System.out.println("获取绝对路径："+ file.getPath());
		System.out.println("获取绝对路径："+ file.getAbsolutePath());
		System.out.println("获取文件的大小（字节为单位）:"+file.length());
		System.out.println("获取父路径:"+ file.getParent());
		long time = file.lastModified(); //获取文件最后的修改时间，返回的是一个毫秒值。
		Date date = new Date(time);
		//日期格式化类
		SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy年MM月dd日   HH:mm:ss");
		System.out.println("最后的修改时间："+ dateFormat.format(date));
	}
}
```

## 文件夹相关

``` java
/*
文件夹相关：
staic File[] listRoots()	列出所有的根目录（Window中就是所有系统的盘符）

list()						返回目录下的文件或者目录名，包含隐藏文件。对于文件这样操作会返回null。
listFiles()					返回目录下的文件或者目录对象（File类实例），包含隐藏文件。对于文件这样操作会返回null。
*/
public class Demo7 {
	public static void main(String[] args) {
		/*File[] files = File.listRoots();   //列出所有的盘符
		for(File file : files){
			System.out.println(file);
		}*/
		
		File file = new File("F:\\0416\\day01");
		/*String[] fileNames = file.list(); //获取当前路径下面的所有子文件名与子文件夹名。 
		for(String fileName : fileNames){
			System.out.println(fileName);
		}*/
		
		File[] files = file.listFiles();  //把子文件与子目录存储到一个数组中返回。
		for(File fileItem : files){
			System.out.println(fileItem.getName());
		}
	}
}
```

## 条件过滤

``` java
/*
1。 定义一个函数列出指定目录中所有的java文件。
2，列出指定目录中所有的子文件名与所有的子目录名，要求目录名与文件名分开列出，格式如下：
		子目录：
			jdk
			视频
			代码
		子文件：
			...
			...
list()
listFiles();

list(FilenameFilter filter)	返回指定当前目录中符合过滤条件的子文件或子目录。对于文件这样操作会返回null。
listFiles(FilenameFilter filter)	返回指定当前目录中符合过滤条件的子文件或子目录。对于文件这样操作会返回null。

*/
//自定义一个文件名过滤器
class JavaFileFilter implements FilenameFilter{

	@Override
	/*
	 *  dir  当前文件所在的目录
	 *   name 当前文件的文件名。 
	 */
	public boolean accept(File dir, String name) {
		return name.endsWith(".java");
	}
	
}

public class Demo8 {
	
	public static void main(String[] args) {
		File dir = new File("E:\\workspace\\mybatis\\generatorSqlmapCustom\\src\\com\\it\\dao");
		/*listJava(dir);
		list(dir);
		*/
		listJava2(dir);
	}
	
	public static void listJava2(File dir){
		File[] files = dir.listFiles(new JavaFileFilter()); //返回所有符合条件的子文件与子目录
		for(File file : files){
			System.out.println(file.getName());
		}
	}
	
	public static void list(File dir){
		//获取到当前路径下的所有子文件与子目录
		File[] files = dir.listFiles();
		
		System.out.println("子文件:");
		for(File file : files){
			if(file.isFile()){		
				System.out.println("\t"+file.getName());
			}
		}
		
		System.out.println("子目录：");
		for(File file : files){
			if(file.isDirectory()){
				System.out.println("\t"+file.getName());
			}
		}
	}
	
	//列出所有的java文件
	public static void listJava(File dir){
		File[] files =dir.listFiles() ; //找到了目录下面的所有子文件
		for(File file : files){
			String fileName = file.getName(); //获取到了文件名
			if(fileName.matches(".+\\.java")){
				System.out.println(fileName);
			}
		}
	}

}
```