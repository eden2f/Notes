# Java 异常处理中在finally里面写return会怎么样

> 博客原文引用 : [https://blog.csdn.net/u011277123/article/details/59074492](https://blog.csdn.net/u011277123/article/details/59074492)

java里面异常模块中finally是肯定会执行的，那么，如果在finally语句块中写了return语句了，会怎么样呢？下面就来试试~~
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320232158.png#id=zzb5U&originHeight=269&originWidth=597&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 程序方法
首先第一句的@SuppressWarnings("finally")是一个批注，该批注的作用是给编译器一条指令，告诉它对被批注的代码元素内部的某些警告保持静默。其实就是在finally里面加return语句的话，编译器会发出警告，说"最后块不正常完成"，所以强迫症看着有些不习惯，就用批注把警告弄掉了。下面看下代码，在传值进来后，对值进行判断，如果值小于0了，我们就抛一个数据格式错误的异常出去，同时在catch里面再抛出去。如果大于0，那么久return出去，然后看下finally里面，return一个-1出去；
那么，我们测试的时候传两个值进去，一个是-1一个是100，运行看下结果：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320232413.png#id=PVU47&originHeight=324&originWidth=635&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 测试结果
可以看到不管是-1（-1只是测试数据和finally返回的-1不是同一个概念，换成-2也是输出的-1）还是100，输出都是-1 。那么这是为什么呢，为什么100的时候return出去却无效了呢？还有一二问题，明明在doIt（）方法里抛出了异常，在main里面try，catch了，为什么捕捉不到异常呢？
先解答第一个：
因为finally里面写了return语句的时候，就会覆盖掉try代码块里面的return。因为finally是肯定会执行的，所以，当传进-1时，捕捉到异常后直接抛出异常，之后执行到finally代码块，finally中把返回值给重置了，所以，返回的是-1的值。同理100的情况跟这个也差不多，我们用Debug运行查看时，可以看到如果不满足小于0的条件时，直接就跳到了finally块去执行return了。这是为什么我们在finally中写return后程序会给我们警告，警告的内容也就是上面所说的“最后块不正常完成”。那么，问题又来了，可以在finally里面去修改变量的值，然后再return出去呢？
修改变量的方法
那么，大家猜猜这个代码返回的会是什么呢？
该方法的返回值永远都是2.而不会是-1或者0，那为什么不会执行到finally后面的“return 0”呢，因为在finally执行完毕之后该方法就已经有返回值了，后续代码就不会再执行了（大家也可以用Debug调试试一下），然后后面的修改值无效是因为前面return的时候，已经确定了return出去的是int类型，那么由于基本类型都是值的拷贝，所以finally后面再修改a的值已经没有意义了，所以返回的永远都是2.
接着解释第二个：为什么把异常throw出去了，在mian里面却捕捉不到，因为异常线程在监视到有异常发生的时候，就会登记当前的异常类型为DataFormatException，但是当执行器执行finally代码块的时候，重新为doIt方法赋值了，也就是告诉了线程这个这个方法没有异常，所以就相当于把异常给屏蔽了，这样在mian里面也就无法捕捉到异常了。
所以讲这么多，其实是为了告诉大家，千万不要在finally语句里面处理返回值什么的，因为一写下去，会产生一些很隐蔽的错误，写代码的时候要切忌~~~~
好了今天就到这边了谢谢观看，喜欢的关注一波持续更新中
## 初步结论
1、不管有木有出现异常，finally块中代码都会执行；
2、当try和catch中有return时，finally仍然会执行；
3、finally是在return后面的表达式运算后执行的（此时并没有返回运算后的值，而是先把要返回的值保存起来，管finally中的代码怎么样，返回的值都不会改变，任然是之前保存的值），所以函数返回值是在finally执行前确定的；
4、finally中最好不要包含return，否则程序会提前退出，返回值不是try或catch中保存的返回值。
## 举例
情况1：try{} catch(){}finally{} return; 显然程序按顺序执行。
情况2:try{ return; }catch(){} finally{} return; 程序执行try块中return之前（包括return语句中的表达式运算）代码； 再执行finally块，最后执行try中return; finally块之后的语句return，因为程序在try中已经return所以不再执行。
情况3:try{ } catch(){return;} finally{} return; 程序先执行try，如果遇到异常执行catch块， 有异常：则执行catch中return之前（包括return语句中的表达式运算）代码，再执行finally语句中全部代码， 最后执行catch块中return. finally之后也就是4处的代码不再执行。 无异常：执行完try再finally再return.
情况4:try{ return; }catch(){} finally{return;} 程序执行try块中return之前（包括return语句中的表达式运算）代码； 再执行finally块，因为finally块中有return所以提前退出。
情况5:try{} catch(){return;}finally{return;} 程序执行catch块中return之前（包括return语句中的表达式运算）代码； 再执行finally块，因为finally块中有return所以提前退出。
情况6:try{ return;}catch(){return;} finally{return;} 程序执行try块中return之前（包括return语句中的表达式运算）代码； 有异常：执行catch块中return之前（包括return语句中的表达式运算）代码； 则再执行finally块，因为finally块中有return所以提前退出。 无异常：则再执行finally块，因为finally块中有return所以提前退出。
## 最终结论
任何执行try 或者catch中的return语句之前，都会先执行finally语句，如果finally存在的话。 如果finally中有return语句，那么程序就return了，所以finally中的return是一定会被return的。
