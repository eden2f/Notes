# Java线程 死锁

## 死锁没有办法解决，只能避免

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122005125.png)

## 根本原因

java同步机制解决了线程安全问题，但是同时也引发了死锁现象。

* 死锁现象如何解决呢？
  * 没法解决。 只能尽量的避免死锁现象。
* 死锁现象出现的根本原因
  1. 存在两个或者两个以上的线程存在。
  2. 多个线程必须共享两个或者两个以上的资源。

## 示例

``` java
class DeadLockThread extends Thread{
	public DeadLockThread(String name){
		super(name);
	}
	@Override
	public void run() {
		if("张三".equals(this.getName())){
			synchronized ("遥控器") {
				System.out.println(this.getName()+"取走了遥控器，准备取电池");
				synchronized ("电池") {
					System.out.println(this.getName()+"取到了电池，开着空调爽歪歪的吹着 ！！");
				}
			}
		}else if("李四".equals(this.getName())){
			synchronized ("电池") {
				System.out.println(this.getName()+"取走了电池，准备取取遥控器");
				synchronized ("遥控器") {
					System.out.println(this.getName()+"取走了遥控器，开着空调爽歪歪的吹着 ！！");
				}
			}
		}
	}
}
public class Demo2 {
	public static void main(String[] args) {
		//创建了线程对象
		DeadLockThread thread1 = new DeadLockThread("张三");
		DeadLockThread thread2 = new DeadLockThread("李四");
		thread1.setPriority(10);
		thread2.setPriority(1);
		//调用start方法启动线程
		thread1.start();
		thread2.start();
	}
}
```