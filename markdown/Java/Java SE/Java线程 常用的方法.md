# Java线程 常用的方法

## 线程常用的方法
1. Thread(String name)     初始化线程的名字
2. setName(String name)    设置线程对象名
3. getName()               返回线程的名字
4. static sleep()                 由于是 static 方法，所以是对当前的正在运行的线程生效。那个线程执行了sleep的代码 ，那么该线程就会睡眠指定毫秒数
5. currentThread()      	返回当前执行该方法的线程对象引用
6. getPriority()             返回当前线程对象的优先级，默认线程的优先级是5
7. setPriority(int newPriority) 设置线程的优先级    虽然设置了线程的优先级，但是具体的实现取决于底层的操作系统的实现（最大的优先级是10 ，最小的1 ， 默认是5）

## 示例
``` java
public class Demo3 extends Thread {
	public Demo3(String name){
		super(name);// 指定调用Thread类一个参数的构造方法。给线程初始化名字。
	}
	@Override
	public void run() {
		for(int i = 0 ; i<100 ; i++){
			System.out.println(Thread.currentThread().getName()+":"+i);
		}
		//System.out.println(Thread.currentThread()==this);
	}
	public static void main(String[] args) throws Exception {
		//创建一个自定义的线程对象
		Demo3 d = new Demo3("狗娃");
		d.setPriority(1);
		d.start();
		System.out.println("自定义线程的优先级："+ d.getPriority());

//		Thread.sleep(1000);  指定线程睡眠的毫秒数
		Thread mainThread = Thread.currentThread() ; // 返回当前线程.
		System.out.println("主线程的优先级："+ mainThread.getPriority());   //默认的优先级是5 .
		mainThread.setPriority(10);  //设置线程的优先级  优先级越高的线程得到cpu的概率越大。    优先级的范围：１～１０　
		System.out.println("主线程的名字："+ mainThread.getName());

		for(int i = 0 ; i<100 ; i++){
			System.out.println(mainThread.getName()+":"+i);
		}
	}
}

```