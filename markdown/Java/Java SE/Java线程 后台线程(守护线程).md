# Java线程 后台线程(守护线程)


## 后台线程

就是隐藏起来一直在默默运行的线程，直到进程结束。

## 实现

setDaemon(boolean on)

## 特点

* 当所有的非后台线程结束时，程序也就终止了同时还会杀死进程中的所有后台线程，也就是说，只要有非后台线程还在运行，程序就不会终止，执行main方法的主线程就是一个非后台线程。
* 必须在启动线程之前（调用start方法之前）调用setDaemon（true）方法，才可以把该线程设置为后台线程。
* 一旦main（）执行完毕，那么程序就会终止，JVM也就退出了。
* 可以使用isDaemon() 测试该线程是否为后台线程（守护线程）。

## 示例

``` java
/*
守护线程(后台线程) : 当前一个java应用只剩下守护线程的时候，那么守护线程马上结束。
守护线程应用场景：
	1. 新的软件版本下载。
 需求：模拟QQ在下载更新包.
 守护线程要注意的事项：
 	1. 所有的线程默认都不是守护线程。
 */
public class Demo5 extends Thread {
	public Demo5(String name){
		super(name);
	}
	@Override
	public void run()  { // 子类抛出的异常类型必须要小于或者等于父类抛出 的异常类型。
		for(int i = 1 ;  i<100 ; i++){
			System.out.println(this.getName()+"已经下载了："+i+"%");
			try {
				Thread.sleep(100);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		System.out.println("下载完毕，正在安装更新包！！！");
	}
	public static void main(String[] args) {
		//创建一个线程对象
		Demo5 d = new Demo5("守护线程");

		d.setDaemon(true); //设置一个线程为守护线程。
		System.out.println("是守护线程吗？"+d.isDaemon());  // isDaemon 判断一个线程是否为守护线程。
		//启动线程
		d.start();
		for(int i = 0; i < 100 ; i++){
			System.out.println(Thread.currentThread().getName()+":"+i);
		}
	}
}
```