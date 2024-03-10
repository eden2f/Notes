# SpringBoot 应用启动原理(一) 将启动脚本嵌入Jar

> [segmentfault 博客原文引用](https://segmentfault.com/a/1190000013489340)
> [Spring Boot](https://projects.spring.io/spring-boot/)
> Takes an opinionated view of building production-ready Spring applications. Spring Boot favors convention over configuration and is designed to get you up and running as quickly as possible.

SpringBoot项目为我们构建Spring应用带来了极大的方便，同时SpringBoot在构建Spring应用方面也做出了很大建树
众所周知，SpringBoot可以通过gradle或者maven插件构建Executable Jar/War [Spring Boot Gradle Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/gradle-plugin/reference/html/#packaging-executable-wars)
除了传统方式`java -jar myapp.jar`运行外，还可以通过`myapp.jar start|stop|restart`运行，安装为systemd服务，通过同名文件myapp.conf配置运行时参数等等高级功能 [Installing Spring Boot Applications](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/reference/html/deployment-install.html)
讲到这里很多童鞋都会问，这一切都是如何做到的？
## 将启动脚本嵌入jar
首先，我们创建一个简单的示例
```
package com.manerfan.springboot.theory;

/**
 * @author manerfan
 * @date 2018/3/2
 */

public class RunnableApp {
    public static void main(String[] args) {
        System.out.println("Hello You!");
    }
}
```
使用eclipse或idea或其他工具，编译并打包为jar（spring-boot-theory.jar），打包时选择main-class为com.manerfan.SpringBoot.theory.RunnableApp
对于Runnable Jar，总有一个META-INF/MANIFEST.MF文件，记录Main-Class、Class-Path等信息
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223113.png#id=yZb6m&originHeight=305&originWidth=732&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们可以通过`java -jar spring-boot-theory.jar`来运行，但尝试直接运行`spring-boot-theory.jar`时便会报错
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223146.png#id=lbdZ3&originHeight=88&originWidth=351&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这表明，`spring-boot-theory.jar`仅仅为Runnable Jar，而不是Executable Jar
### 使用shell脚本启动jar
一般情况，我们都会借助shell脚本来运行我们的jar，如下 `runJar.sh`
```
#!/bin/sh
JAR="/usr/local/spring-boot-theory.jar"
java=java
if test -n "$JAVA_HOME"; then
    java="$JAVA_HOME/bin/java"
fi
exec "$java" -jar $JAR "$@"
exit 1
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223230.png#id=LxmDr&originHeight=51&originWidth=237&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在此基础上，我们可以加入更多控制，以实现runJar.sh **start | stop | restart*等特性
```
#!/bin/bash  
# chkconfig: 2345 85 85  
# description: spring boot theory
# processname: spring-boot-theory  
# Created By: manerfan (manerfan.china@gmail.com)  

JAR="/usr/local/spring-boot-theory.jar"
PIDFILE=/data/sms-service/smss.pid  
java=java
if test -n "$JAVA_HOME"; then
    java="$JAVA_HOME/bin/java"
fi

start() {}
stop() {}
restart() {}
status() {}

case "$action" in
start)
  start "$@"; exit $?;;
stop)
  stop "$@"; exit $?;;
restart)
  restart "$@"; exit $?;;
status)
  status "$@"; exit $?;;
*)
  echo "Usage: $0 {start|stop|force-stop|restart|force-reload|status|run}"; exit 1;
esac

exit 0
```
可以参考 [http://blog.csdn.net/zhanngle...](http://blog.csdn.net/zhanngle/article/details/51437768)
但这样也只是通过shell脚本控制jar的启动停止，如何做到Executable Jar呢？
### 整合shell脚本与Runnable Jar
同样，首先是一段shell脚本 runJar.sh
```
#!/bin/sh
JAR =`which "$0" 2>/dev/null`
[ $? -gt 0 -a -f "$0" ] && JAR="./$0"
java=java
if test -n "$JAVA_HOME"; then
    java="$JAVA_HOME/bin/java"
fi
exec "$java" -jar $JAR "$@"
exit 1
```
通过以下语句将shell脚本与jar文件整合到一起 `~划重点~`
```
cat runJar.sh spring-boot-theory.jar > exec-spring-boot-theory.jar && chmod +x exec-spring-boot-theory.jar
```

大功告成！
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223417.png#id=afR67&originHeight=47&originWidth=732&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
同样，在此基础上，我们可以加入更多控制，以实现exec-spring-boot-theory.jar **start | stop | restart*等特性
可以参考 [https://coderwall.com/p/ssuax...](https://coderwall.com/p/ssuaxa/how-to-make-a-jar-file-linux-executable)
## Spring Boot的实现原理
SpringBoot项目源码在[https://github.com/spring-pro...](https://github.com/spring-projects/spring-boot)，可以对照查看
我们从 JarWriter 开始
```
public JarWriter(File file, LaunchScript launchScript)
    throws FileNotFoundException, IOException {
  FileOutputStream fileOutputStream = new FileOutputStream(file);
  if (launchScript != null) {
    // 将启动脚本写入文件
    fileOutputStream.write(launchScript.toByteArray());
    // 设置文件可执行属性
    setExecutableFilePermission(file);
  }
  this.jarOutput = new JarArchiveOutputStream(fileOutputStream);
  this.jarOutput.setEncoding("UTF-8");
}
```
当执行gradle build或mvn package时，会使用JarWriter重新生成jar文件。JarWrite构造函数中，会首先将启动脚本写入文件，并设置文件的可执行属性。
除此之外，JarWriter还有众多方法，如writeManifest写入manifest文件、writeNestedLibrary写入第三方依赖等等，通过JarWriter以构建Executable Jar.
此过程，与上述将shell脚本与jar文件整合效果一致。
但是，launchScript又是什么？
```
public DefaultLaunchScript(File file, Map<?, ?> properties) throws IOException {
  // 加载启动脚本
  String content = loadContent(file);
  this.content = expandPlaceholders(content, properties);
}

private String loadContent(File file) throws IOException {
  if (file == null) {
    // 默认launch.script
    return loadContent(getClass().getResourceAsStream("launch.script"));
  }
  return loadContent(new FileInputStream(file));
}
```
默认的LaunchScript为DefaultLaunchScript，在构造DefaultLaunchScript时，若不指定启动脚本，则取默认的launch.script，内容见 [launch.script](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader-tools/src/main/resources/org/springframework/boot/loader/tools/launch.script)
launch.script实现较为复杂，此处不做解析，launch.script与上述shell脚本的实现思路基本相同，同样实现了start stop restart等功能，方便安装为systemd服务
不同的是，launch.script会解析与jar文件同名的conf文件，以实现启动脚本定制化 [Customizing a Script When It Runs](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/reference/html/deployment-install.html#deployment-script-customization-when-it-runs)
如，我们实现一个简单的web接口
```
@SpringBootApplication
@RestController
public class WebApp {
    public static void main(String[] args) {
        SpringApplication.run(WebApp.class, args);
    }

    @RequestMapping("/")
    @GetMapping
    public String hello() {
        return "Hello You!";
    }
}
```
使用spring-boot-gradle-plugin插件打包，执行`./spring-boot-theory-1.0.0.jar`，可以看到输出
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223518.png#id=cncCs&originHeight=44&originWidth=732&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
访问 [http://localhost](http://localhost/):8080 可以看到 Hello You! 字样
若要对启动参数，如监听端口做修改，除了使用`java -jar spring-boot-theory-1.0.0.jar --server.port=8000`外，还可以新建同名文件 `spring-boot-theroy-1.0.0.conf`，填入内容
```
RUN_ARGS="--server.port=8000"
```
再次执行`./spring-boot-theory-1.0.0.jar`
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223602.png#id=xs29L&originHeight=41&originWidth=732&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
监听端口由默认的8080变为指定的8000
conf配置文件可配置的内容较多，如使用JAVA_OPTS配置jvm运行参数，使用`MODE=service`可将程序放入后台运行等等 [Customizing a Script When It Runs](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/reference/html/deployment-install.html#deployment-script-customization-when-it-runs)
以如下conf配置为例
```
MODE=service
JAVA_OPTS="-Xms1g -Xmx1g -Dfile.encoding=utf-8"
RUN_ARGS="--server.port=8000"
```
执行`./spring-boot-theory-1.0.0.jar start`
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223638.png#id=FZRQt&originHeight=107&originWidth=671&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
查看该进程运行参数
```
/usr/bin/java -Dsun.misc.URLClassPath.disableJarChecking=true -Xms1g -Xmx1g -Dfile.encoding=utf-8 -jar /Users/manerfan/Project/learning/javaspring-boot-theory/build/libs/spring-boot-theory-1.0.0.jar --server.port=8000
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240310223713.png#id=WZC2R&originHeight=36&originWidth=732&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 总结
SpringBoot实现ExecutableJar的原理，便是将启动脚本及原有的jar文件（以及第三方依赖包）写入同一个文件，并给该文件赋可执行权限，结合conf配置文件，使RunnableJar变为ExecutableJar的同时，得以更加便捷的控制程序的启动/运行参数。
