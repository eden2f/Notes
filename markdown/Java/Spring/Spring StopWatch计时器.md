# Spring StopWatch计时器

[原文链接跳转 -> 一个不二](https://blog.csdn.net/gxs1688/article/details/87185030)
## StopWatch简介
StopWatch是位于org.springframework.util包下的一个工具类，通过它可方便的对程序部分代码进行计时(ms级别)，适用于同步单线程代码块。
当然，commons工具包下也有的实现可以直接使用 (org.apache.commons.lang3.time.StopWatch) ，功能差不多，实现也简单，本处就不细说了。
## StopWatch的基本使用
正常情况下，我们如果需要看某段代码的执行耗时，会通过如下的方式进行查看：
```java
public static void main(String[] args) throws InterruptedException {
     StopWatchTest.test0();
//        StopWatchTest.test1();
}

public static void test0() throws InterruptedException {
     long start = System.currentTimeMillis();
     // do something
     Thread.sleep(100);
    long end = System.currentTimeMillis();
    long start2 = System.currentTimeMillis();
    // do something
    Thread.sleep(200);
    long end2 = System.currentTimeMillis();
    System.out.println("某某1执行耗时:" + (end - start));
    System.out.println("某某2执行耗时:" + (end2 - start2));
}
1234567891011121314151617
运行结果：
某某1执行耗时:105
某某2执行耗时:203
123
```
该种方法通过获取执行完成时间与执行开始时间的差值得到程序的执行时间，简单直接有效，但想必写多了也是比较烦人的，尤其是碰到不可描述的代码时，会更加的让人忍不住多写几个bug聊表敬意，而且该结果也不够直观，此时会想是否有一个工具类，提供了这些方法，或者自己写个工具类，刚好可以满足这种场景，并且把结果更加直观的展现出来。
 首先我们的需求如下：

1. 记录开始时间点
2. 记录结束时间点
3. 输出执行时间及各个时间段的占比

根据该需求，我们可直接使用org.springframework.util包下的一个工具类StopWatch,通过该工具类，我们对上述代码做如下改造：
```java
public static void main(String[] args) throws InterruptedException {
//        StopWatchTest.test0();
     StopWatchTest.test1();
}

public static void test1() throws InterruptedException {
     StopWatch sw = new StopWatch("test");
     sw.start("task1");
     // do something
    Thread.sleep(100);
    sw.stop();
    sw.start("task2");
    // do something
    Thread.sleep(200);
    sw.stop();
    System.out.println("sw.prettyPrint()~~~~~~~~~~~~~~~~~");
    System.out.println(sw.prettyPrint());
}
123456789101112131415161718
运行结果：
sw.prettyPrint()~~~~~~~~~~~~~~~~~
StopWatch 'test': running time (millis) = 308
-----------------------------------------
ms     %     Task name
-----------------------------------------
00104  034%  task1
00204  066%  task2
12345678
```
start开始记录，stop停止记录，然后通过StopWatch的prettyPrint方法，可直观的输出代码执行耗时，以及执行时间百分比，瞬间感觉比之前的方式高大上了一个档次。
 除此之外，还有以下两个方法shortSummary,getTotalTimeMillis，查看程序执行时间。
运行代码及结果：
```java
System.out.println("sw.shortSummary()~~~~~~~~~~~~~~~~~");
System.out.println(sw.shortSummary());
System.out.println("sw.getTotalTimeMillis()~~~~~~~~~~~~~~~~~");
System.out.println(sw.getTotalTimeMillis());
运行结果
sw.shortSummary()~~~~~~~~~~~~~~~~~
StopWatch 'test': running time (millis) = 308
sw.getTotalTimeMillis()~~~~~~~~~~~~~~~~~
308
123456789
```
其实以上内容在该工具类中实现也极其简单，通过start与stop方法分别记录开始时间与结束时间，其中在记录结束时间时，会维护一个链表类型的tasklist属性，从而使该类可记录多个任务，最后的输出也仅仅是对之前记录的信息做了一个统一的归纳输出，从而使结果更加直观的展示出来。
## StopWatch优缺点
优点：

1. spring自带工具类，可直接使用
2. 代码实现简单，使用更简单
3. 统一归纳，展示每项任务耗时与占用总时间的百分比，展示结果直观性能消耗相对较小，并且最大程度的保证了start与stop之间的时间记录的准确性
4. 可在start时直接指定任务名字，从而更加直观的显示记录结果

缺点：

1. 一个StopWatch实例一次只能开启一个task，不能同时start多个task，并且在该task未stop之前不能start一个新的task，必须在该task stop之后才能开启新的task，若要一次开启多个，需要new不同的StopWatch实例
2. 代码侵入式使用，需要改动多处代码
## Spring中StopWatch源码实现
```java
import java.text.NumberFormat;
import java.util.LinkedList;
import java.util.List;

public class StopWatch {
    private final String id;
    private boolean keepTaskList = true;
    private final List<TaskInfo> taskList = new LinkedList();
    private long startTimeMillis;
    private boolean running;
    private String currentTaskName;
    private StopWatch.TaskInfo lastTaskInfo;
    private int taskCount;
    private long totalTimeMillis;

    public StopWatch() {
        this.id = "";
    }

    public StopWatch(String id) {
        this.id = id;
    }

    public void setKeepTaskList(boolean keepTaskList) {
        this.keepTaskList = keepTaskList;
    }

    public void start() throws IllegalStateException {
        this.start("");
    }

    public void start(String taskName) throws IllegalStateException {
        if (this.running) {
            throw new IllegalStateException("Can't start StopWatch: it's already running");
        } else {
            this.startTimeMillis = System.currentTimeMillis();
            this.running = true;
            this.currentTaskName = taskName;
        }
    }

    public void stop() throws IllegalStateException {
        if (!this.running) {
            throw new IllegalStateException("Can't stop StopWatch: it's not running");
        } else {
            long lastTime = System.currentTimeMillis() - this.startTimeMillis;
            this.totalTimeMillis += lastTime;
            this.lastTaskInfo = new StopWatch.TaskInfo(this.currentTaskName, lastTime);
            if (this.keepTaskList) {
                this.taskList.add(this.lastTaskInfo);
            }

            ++this.taskCount;
            this.running = false;
            this.currentTaskName = null;
        }
    }

    public boolean isRunning() {
        return this.running;
    }

    public long getLastTaskTimeMillis() throws IllegalStateException {
        if (this.lastTaskInfo == null) {
            throw new IllegalStateException("No tasks run: can't get last task interval");
        } else {
            return this.lastTaskInfo.getTimeMillis();
        }
    }

    public String getLastTaskName() throws IllegalStateException {
        if (this.lastTaskInfo == null) {
            throw new IllegalStateException("No tasks run: can't get last task name");
        } else {
            return this.lastTaskInfo.getTaskName();
        }
    }

    public StopWatch.TaskInfo getLastTaskInfo() throws IllegalStateException {
        if (this.lastTaskInfo == null) {
            throw new IllegalStateException("No tasks run: can't get last task info");
        } else {
            return this.lastTaskInfo;
        }
    }

    public long getTotalTimeMillis() {
        return this.totalTimeMillis;
    }

    public double getTotalTimeSeconds() {
        return (double) this.totalTimeMillis / 1000.0D;
    }

    public int getTaskCount() {
        return this.taskCount;
    }

    public StopWatch.TaskInfo[] getTaskInfo() {
        if (!this.keepTaskList) {
            throw new UnsupportedOperationException("Task info is not being kept!");
        } else {
            return (StopWatch.TaskInfo[]) this.taskList.toArray(new StopWatch.TaskInfo[this.taskList.size()]);
        }
    }

    public String shortSummary() {
        return "StopWatch '" + this.id + "': running time (millis) = " + this.getTotalTimeMillis();
    }

    public String prettyPrint() {
        StringBuilder sb = new StringBuilder(this.shortSummary());
        sb.append('\n');
        if (!this.keepTaskList) {
            sb.append("No task info kept");
        } else {
            sb.append("-----------------------------------------\n");
            sb.append("ms     %     Task name\n");
            sb.append("-----------------------------------------\n");
            NumberFormat nf = NumberFormat.getNumberInstance();
            nf.setMinimumIntegerDigits(5);
            nf.setGroupingUsed(false);
            NumberFormat pf = NumberFormat.getPercentInstance();
            pf.setMinimumIntegerDigits(3);
            pf.setGroupingUsed(false);
            StopWatch.TaskInfo[] var7;
            int var6 = (var7 = this.getTaskInfo()).length;

            for (int var5 = 0; var5 < var6; ++var5) {
                StopWatch.TaskInfo task = var7[var5];
                sb.append(nf.format(task.getTimeMillis())).append("  ");
                sb.append(pf.format(task.getTimeSeconds() / this.getTotalTimeSeconds())).append("  ");
                sb.append(task.getTaskName()).append("\n");
            }
        }

        return sb.toString();
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder(this.shortSummary());
        if (this.keepTaskList) {
            StopWatch.TaskInfo[] var5;
            int var4 = (var5 = this.getTaskInfo()).length;

            for (int var3 = 0; var3 < var4; ++var3) {
                StopWatch.TaskInfo task = var5[var3];
                sb.append("; [").append(task.getTaskName()).append("] took ").append(task.getTimeMillis());
                long percent = Math.round(100.0D * task.getTimeSeconds() / this.getTotalTimeSeconds());
                sb.append(" = ").append(percent).append("%");
            }
        } else {
            sb.append("; no task info kept");
        }

        return sb.toString();
    }

    public static final class TaskInfo {
        private final String taskName;
        private final long timeMillis;

        TaskInfo(String taskName, long timeMillis) {
            this.taskName = taskName;
            this.timeMillis = timeMillis;
        }

        public String getTaskName() {
            return this.taskName;
        }

        public long getTimeMillis() {
            return this.timeMillis;
        }

        public double getTimeSeconds() {
            return (double) this.timeMillis / 1000.0D;
        }
    }

}
```
