# Java线程 synchronized同步代码块

## 线程安全问题的产生的原因

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122003103.png)

## 线程安全问题的解决方案

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240122003132.png)

## 需求 : 模拟车站卖50张票

### 问题1 : 50张票被卖了150次 ?

* 原因 : num是非静态的成员变量,非静态的成员变量每创建一个SaleTickets对象的时候，都会在内部维护一份num数据，这时候创建了三个SaleTickets对象，所以就有三份num数据。
* 解决方案 : 使用static修饰票数num，让该数据共享出来给所有的对象使用。

### 问题2 : 出现了线程安全问题。

* 线程安全问题出现的根本原因：
    * 存在着两个或者两个以上的线程。
    * 多个线程共享了着一个资源， 而且操作资源的代码有多句。

## 线程安全问题的解决方案

### 使用同步代码块

* 格式：
    ``` java
        synchronized(锁对象){
            需要被同步的代码；
        }
    ```
#### 同步代码块要注意的细节：

1. 锁对象可以是任意的对象、.
2. 锁对象必须是多个线程共享的对象（锁对象必须是唯一）。
3. 线程调用了sleep方法是不会释放锁对象的。
4. 只有会出现线程安全问题的时候才使用java的同步机制（同步代码块和同步 函数）

### 同步函数

## 示例

``` java
/*
练习：模拟车站卖50张票。  不准出现线程安全问题。
 */
class SaleTickets extends Thread{
	static	int num = 50;  //非静态成员变量。 非静态成员变量在每个对象中都维护了一份数据。
	static	Object o = new Object();
	public SaleTickets(String name){
		super(name); //调用父类一个参数的构造函数， 初始化线程的名字。
	}
	//线程的任务代码...
	@Override
	public void run() {
		while(true){
			synchronized ("锁") {
				if(num>0){
					try {
						Thread.sleep(100);
						System.out.println(Thread.currentThread().getName()+"卖出了"+num+"号票");
						num--;
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}else{
					System.out.println("售罄了...");
					break;
				}
			}
		}
	}
}
public class Demo4 {
	public static void main(String[] args) {
		//创建线程对象
		SaleTickets thread1 = new SaleTickets("窗口1");
		SaleTickets thread2 = new SaleTickets("窗口2");
		SaleTickets thread3 = new SaleTickets("窗口3");
		//开启线程
		thread1.start();
		thread2.start();
		thread3.start();
	}
}
```