# JavaIO File的引入

## IO流技术

解决设备与设备之间的数据传输问题。    

比如：内存----->硬盘,硬盘------>内存,键盘----->内存

## IO技术的应用场景

导出报表,上传大头照,播放音频文件,切水果…… 

如果数据想永久性的保存起来，那么数据一般会保存在硬盘上，硬盘的数据一般以文件形式存在

Sun也使用了一个类描述了文件与文件-----File

## File的构造函数

* File(String pathname)   指定文件或者文件夹的路径，创建一个File对象 
* File(File parent, String child)   指定父路径与子路径构建一个File对象 。 
* File(String parent, String child)     指定父路径与子路径构建一个File对象 

## 示例

``` java
public class Demo1 {
	public static void main(String[] args) {
		File parentFile = new File("F:\\");   //有时候我们是需要父路径先做预处理之后， 然后我们才能对子文件进行另外的处理。
		File file  = new File("F:\\","a.txt");
		System.out.println("存在吗？"+ file.exists());
	}
}
```