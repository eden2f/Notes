# Java线程 线程的通讯

## 线程的通讯

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122010015.png)

当一个线程完成了一个任务的时候，要通知另外一个线程去处理其他 的事情。

问题1： 价格错乱.. （线程安全问题）

问题2：生产一个、消费一个。
	
## 线程通讯的方法

* wait()    执行了wait方法的线程，会让该线程进入以锁对象建立的线程池中等待。
* notify()    如果一个线程执行了notify方法，该线程会唤醒以锁对象建立的线程池中等待线程中的一个.
* notifyAll();     把所有的线程都唤醒。()
  
## 线程通讯要注意的事项

1. wait、 notify、 notifyAll方法都是属于Object对象的方法。
2. wait、notify方法必须要在同步代码块或者是同步函数中调用。
3. wait、notify方法必须由锁对象调用，否则报错。
4. 一个线程执行了 wait 方法会释放资源，释放锁。是Object的方法
5. 一个线程执行了 sleep() 方法会释放资源，不释放锁。是Thread的方法

## 示例

``` java
/**
 * 产品类
 */
class Product{

	String name;

	int price;

	boolean flag ;  //产品是否生成完毕的标识   false为还没有生成完毕， true 生成完毕了.

}

/**
 * 生产者类
 */
class Producer extends Thread{

	//维护一个产品
	Product p;

	public Producer(Product p){
		this.p = p;
	}

	@Override
	public void run() {
		int i = 0;
		while(true){
			synchronized (p) {
				if(p.flag==false){
					if(i%2==0){
						p.name = "摩托车";
						p.price= 4000;
					}else{
						p.name = "自行车";
						p.price = 300;
					}
					System.out.println("生产了"+ p.name+" 价格："+ p.price);
					i++;
					//生成完毕 --- 改标识
					p.flag = true;
					//唤醒消费者去消费
					p.notifyAll();
				}else{
					//如果产品已经生产完毕，应该等待消费者先消费
					try {
						p.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}
		}
	}
}

//消费者
class Customer extends Thread{

	//产品
	Product p;

	public Customer(Product p){
		this.p = p;
	}


	@Override
	public void run() {
		while(true){
			synchronized (p) {
				if(p.flag==true){
					System.out.println("消费者消费了："+ p.name+" 价格："+ p.price);
					//改标识
					p.flag = false;
					p.notifyAll();
				}else{
					//如果产品已经被消费完毕,应该唤醒生产者去生成
					try {
						p.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}
		}
	}
}

public class Demo7 {

	public static void main(String[] args) {
		//创建一个产品对象
		Product p = new Product();
		//创建线程对象
		Producer producer = new Producer(p);
		Customer customer = new Customer(p);
		//启动线程
		producer.start();
		customer.start();

	}
}
```