# Java线程 自定义线程 implements Runnable

## 自定义线程的创建方式

### 方式一

1. 自定义一个类继承Thread.
2. 子类重写run方法，把自定义线程的任务定义在run方法上。
3. 创建thread子类的对象，并且调用start方法开启线程。

### 方式二

1. 自定义一个类去实现Runnable接口。
2. 实现了Runnable接口的run方法， 把自定义线程的任务定义在run方法上。
3. 创建Runnable实现类的对象。
4. 创建Thread对象，并且把Runnable实现类对象作为参数传递进去。
5. 调用thread对象的start方法开启线程。
  
## 疑问1 : Runnable实现类对象是线程对象吗？

* runnable实现类的对象并不是一个线程对象，只不过是实现了Runnable接口的对象而已。
  
## 疑问2 : 为什么要把Runnable实现类的对象作为参数传递给Thread对象呢？作用是什么？
	
* 作用 : 是把Runnable实现类的对象的run方法作为了任务代码去执行了。

## 推荐方式

推荐使用第二种。因为java是单继承的。

## 示例

``` java
public class Demo3 implements Runnable{
	@Override
	public void run() {
		for(int i = 0 ; i< 100 ; i++){
			System.out.println(Thread.currentThread().getName()+":"+i);
		}
		System.out.println("当前线程对象："+Thread.currentThread());  // 当前线程对象是： Thread
		System.out.println("当前对象："+ this);   //this对象： Demo3的对象
	}

	public static void main(String[] args) {
		//创建Runnable实现类的对象
		Demo3 d = new Demo3();
		//创建Thread对象，并且把Runnable实现类对象作为参数传递进去
		Thread t = new Thread(d,"狗娃");
		//调用thead对象的start方法开启线程。
		t.start();

		//主线程执行的。
		for(int i = 0 ; i< 100 ; i++){
			System.out.println(Thread.currentThread().getName()+":"+i);
		}
	}
}
```