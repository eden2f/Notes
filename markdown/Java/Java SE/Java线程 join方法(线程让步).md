# Java线程 join方法(线程让步)

## join方法(线程让步)

当A线程执行到了B线程Join方法时A就会等待，等B线程都执行完A才会执行，Join可以用来临时加入线程执行.

## 需求：模拟小时候打酱油.

``` java
class Mother extends Thread{
	@Override
	public void run() {
		System.out.println("妈妈洗菜...");
		System.out.println("妈妈切菜...");
		System.out.println("妈妈发现没有酱油了...");
		//通知儿子去打酱油
		Son s = new Son();
		s.start();
		try {
			s.join();   //  join 加入  : 如果当前线程执行了join方法，那么当前线程就会让步给新加入的线程先完成任务，然后当前线程才继续的执行自己的任务。

		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("妈妈炒菜...");
		System.out.println("全家一起吃饭...");
	}
}

//儿子线程
class Son extends Thread{

	@Override
	public void run() {
		try {
			System.out.println("儿子下楼梯");
			Thread.sleep(1000);
			System.out.println("儿子一直往前走...");
			System.out.println("儿子买到了酱油...");
			System.out.println("儿子跑回来...");
			Thread.sleep(1000);
			System.out.println("儿子把酱油给老妈..");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

public class Demo6 {

	public static void main(String[] args) {
		Mother m =  new Mother();
		m.start();
	}
}

```