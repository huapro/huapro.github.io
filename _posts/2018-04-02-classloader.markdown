---
layout:     post
title:      "ClassLoader，Thread.currentThread().setContextClassLoader，tomcat的ClassLoader"
subtitle:   extension project
date:       2018-04-02 19:38:20 
author:     "Hua"
header-img: "img/网络.JPG"
catalog: true
multilingual: false
tags:
    - java
---

> “java学习总结 ”


实际上，在Java应用中所有程序都运行在线程里，如果在程序中没有手工设置过ClassLoader，对于一般的java类如下两种方法获得的ClassLoader通常都是同一个 

```
this.getClass.getClassLoader()；  
Thread.currentThread().getContextClassLoader()；  
```

方法一得到的Classloader是静态的，表明类的载入者是谁；

方法二得到的Classloader是动态的，谁执行（某个线程），就是那个执行者的Classloader。对于单例模式的类，静态类等，载入一次后，这个实例会被很多程序（线程）调用，对于这些类，载入的Classloader和执行线程的Classloader通常都不同。

 

## 一、线程上下文类加载器

　　线程上下文类加载器（context class loader）是从 JDK 1.2 开始引入的。类 `java.lang.Thread`中的方法 `getContextClassLoader()`和`setContextClassLoader(ClassLoader cl)`用来获取和设置线程的上下文类加载器。如果没有通过 `setContextClassLoader(ClassLoader cl)`方法进行设置的话，线程将继承其父线程的上下文类加载器。Java 应用运行的初始线程的上下文类加载器是系统类加载器(appClassLoader)。在线程中运行的代码可以通过此类加载器来加载类和资源。

　　前面提到的类加载器的代理模式并不能解决 Java 应用开发中会遇到的类加载器的全部问题。Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。这些 SPI 的接口由 Java 核心库来提供，如 JAXP 的 SPI 接口定义包含在 `javax.xml.parsers`包中。这些 SPI 的实现代码很可能是作为 Java 应用所依赖的 jar 包被包含进来，可以通过类路径（CLASSPATH）来找到，如实现了 JAXP SPI 的 [Apache Xerces](http://xerces.apache.org/)所包含的 jar 包。SPI 接口中的代码经常需要加载具体的实现类。如 JAXP 中的 `javax.xml.parsers.DocumentBuilderFactory`类中的 `newInstance()`方法用来生成一个新的`DocumentBuilderFactory`的实例。这里的实例的真正的类是继承自 `javax.xml.parsers.DocumentBuilderFactory`，由 SPI 的实现所提供的。如在 Apache Xerces 中，实现的类是 `org.apache.xerces.jaxp.DocumentBuilderFactoryImpl`。而问题在于，SPI 的接口是 Java 核心库的一部分，是由引导类加载器来加载的；SPI 实现的 Java 类一般是由系统类加载器来加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能代理给系统类加载器，因为它是系统类加载器的祖先类加载器。也就是说，类加载器的代理模式无法解决这个问题。

　　线程上下文类加载器正好解决了这个问题。如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。线程上下文类加载器在很多 SPI 的实现中都会用到。

**JNDI，JDBC的诉求是：**

**　　**为了能让应用程序访问到这些jar包中的实现类，即用appClassLoarder去加载这些实现类。可以用getContextClassLoader取得当前线程的ClassLoader（即appClassLoarder），然后去加载这些实现类，就能让应用访问到**。**

**tomcat的诉求：**

**　　**稍微跟上面有些不同，容器不希望它下面的webapps之间能互相访问到，所以不能用appClassLoarder去加载。所以tomcat新建一个sharedClassLoader（它的parent是commonClassLoader，commonClassLoader的parent是appClassLoarder，默认情况下，sharedClassLoader和commonClassLoader是同一个UrlClassLoader实例），这是catalina容器使用的ClassLoader。对于每个webapp，为其新建一个webappClassLoader，用于加载webapp下面的类，这样webapp之间就不能相互访问了。tomcat的ClassLoader不完全遵循双亲委派，首先用webappClassLoader去加载某个类，如果找不到，再交给parent。而对于java核心库，不在tomcat的ClassLoader的加载范围。

　　看下tomcat的Bootstrap类的init方法：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public void init() throws Exception {

        initClassLoaders();

        Thread.currentThread().setContextClassLoader(catalinaLoader);//不知道这行设置了之后，对后面有什么用？？？

        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method/*反射实例化Catalina类的实例*/
        Class<?> startupClass =
            catalinaLoader.loadClass
            ("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.newInstance();// Set the shared extensions class loader
        if (log.isDebugEnabled())
            log.debug("Setting startup class properties");
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;

    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　由于Bootstrap类和catalina类被发布在不同包里面，Bootstrap对catalina实例的操作必须用反射完成。

　　catalina类实例（即startupClass）由反射生成，它的ClassLoader是catalinaLoader。然后反射调用方法setParentClassLoader设置catalina类实例里面的变量parentClassLoader为sharedClassLoader，意思是作为容器下webapp的webappClassLoader的parent，而不是设置catalina类的ClassLoader的parent是sharedClassLoader。

　　现在对tomcat的Bootstrap类的init方法里面的Thread.currentThread().setContextClassLoader(catalinaLoader);这一行还是很疑惑。因为，在类catalina里面，可以用getClass().getClassLoader()获取catalinaClassLoader，不需要从Thread.currentThread().getContextClassLoader()方法获得。难道是为了让子线程的ClassLoader都是catalinaClassLoader，而不是appClassLoarder？？

##  

## 二、类加载器与 Web 容器

　　对于运行在 Java EE™容器中的 Web 应用来说，类加载器的实现方式与一般的 Java 应用有所不同。不同的 Web 容器的实现方式也会有所不同。以 Apache Tomcat 来说，每个 Web 应用都有一个对应的类加载器实例。该类加载器也使用代理模式，所不同的是它是首先尝试去加载某个类，如果找不到再代理给父类加载器。这与一般类加载器的顺序是相反的。这是 Java Servlet 规范中的推荐做法，其目的是使得 Web 应用自己的类的优先级高于 Web 容器提供的类。这种代理模式的一个例外是：Java 核心库的类是不在查找范围之内的。这也是为了保证 Java 核心库的类型安全。

　　绝大多数情况下，Web 应用的开发人员不需要考虑与类加载器相关的细节。下面给出几条简单的原则：

- 每个 Web 应用自己的 Java 类文件和使用的库的 jar 包，分别放在 `WEB-INF/classes`和 `WEB-INF/lib`目录下面。
- 多个应用共享的 Java 类文件和 jar 包，分别放在 Web 容器指定的由所有 Web 应用共享的目录下面。
- 当出现找不到类的错误时，检查当前类的类加载器和当前线程的上下文类加载器是否正确。



### 三、ContextClassLoader和其他ClassLoader的关系 

　　我们可以通过getContextClassLoader方法来获得此context classloader，就可以用它来载入我们所需要的Class。默认的是system classloader。
　　bootstrap classloader  -------  对应jvm中某c++写的dll类
　　Extenson ClassLoader ---------对应内部类ExtClassLoader
　　System ClassLoader  ---------对应内部类AppClassLoader
　　Custom ClassLoader  ----------对应任何URLClassLoader的子类（你也可以继承SecureClassLoader或者更加nb一点 直接继承ClassLoader，这样的话你也是神一般的存在了 XD）
　　以上四种classloder按照从上到下的顺序，依次为下一个的parent
　　这个第一概念
　　第二个概念是几个有关的classloader的类
​           抽象类 ClassLoader
​                  |
​            SecureClassLoader
​                   |
​            URLClassloader
​             |           |                
 sun的ExtClassLoader   sun的AppClassLoader
　　以上的类之间是继承关系，与第一个概念说的parent是两回事情，需要小心。
　　第三个概念是Thread的ContextClassLoader
　　其实从Context的名称就可以看出来，这只是一个用以存储任何classloader引用的临时存储空间，与classloader的层次没有任何关系。

 

## 四、Context ClassLoader详解

　　通常情况下，类装载器共有4种，即启动类装载器、EXT类装载器、App类装载器和自定义类装载器。他们之间的阶层情况如下图左面所示，他们都有着不同的载入规则，并且通过向上代理的方式来进行。而本文所提到的Context Class Loader并不是一种新的装载器类型，而是一种抽象的说法，它的具体表现形式为：调用Thread.getCurrentThread().getContextClassLoader()所返回的那个ClassLoader。它和JVM缺省的类装载器以及自定义类装载之间是什么关系呢？下面通过一个实验来看一下。

 ![img](http://p.blog.csdn.net/images/p_blog_csdn_net/kabini/EntryImages/20080924/ContextCL(1).PNG)

 

 

3 实战演练

（1）步骤一

　　上图进行了这样一个实验：首先一个名为Class（1）的类中启动MainThread（其实就是这个类里面有main函数的意思啦），注意这个类的名字后面标出了其所在的路径（即ClassPath），然后在里面进行测试，发现目前它的装载器和当前线程（MainThread）的ContextClassLoader都是AppClassLoader。然后Class（1）启动了一个新线程Class（2）。这里的Class（2）是一个Thread的子类，执行Class（2）代码的线程我称之为Thread-0。

（2）步骤二

　　上图可以看到Class（2）的装载器和ContextClassLoader同样都是AppClassLoader。随后我在Class（2）中创建了一个新的URLCLassLoader，并用这个ClassLoader来载入另一个和Class（1）不在同一个ClassPath下的类Class（3）。此时我们就可以看到变化：即载入Class（3）的装载器是URLClassLoader，而ContextClassLoader还仍然是AppClassLoader。

（2）步骤三

　　最后我们在Class（3）中启动了一个线程类Class（4），发现Class（4）也是由URLClassLoader载入的，而此时ContextClassLoader仍然是AppClassLoader。

​    在整个过程中，装载类的ClassLoader发生了变化，由于线程类Class（4）是由Class（3）启动的，所以装载它的类装载器就变成了URLClassLoader。与此同时，所有线程的ContextClassLoader都继承了生成该线程的ContextClassLoader--AppClassLoader。

 

　　如果我们在第二步的结尾执行了绿色框中的代码：setContextClassLoader()，则结果就会变成下面这个样子：

 ![img](http://p.blog.csdn.net/images/p_blog_csdn_net/kabini/EntryImages/20080924/ContextCL(2).PNG)

　　我们可以清楚地看到，由于Thread-0将其ContextClassLoader设置成了URLClassLoader，而Thread-1是在Thread-0里面生成的，所以就继承了其ContextClassLoader，变成了URLClassLoader。

 

3 后记

　　这里列出的试验可能不见得全面，但相信足以说明问题，应该可以说明ContextClassLoader与其它类装载器的区别所在。但有可能ContextClassLoader还有其他的不同之处，希望有这方面经验的朋友一起讨论。

 

　　Thread.currentThread().getContextClassLoader()的意义:

　　父Classloader可以使用当前线程Thread.currentthread().getContextLoader()中指定的classloader中加载的类。颠覆了父ClassLoader不能使用子Classloader或者是其它没有直接父子关系的Classloader中加载的类这种情况。这个就是Context Class Loader的意义。

 

### **五、Current ClassLoader**

　　当前类所属的ClassLoader，在虚拟机中类之间引用，默认就是使用这个ClassLoader。另外，当你使用Class.forName(), Class.getResource()这几个不带ClassLoader参数的方法时，默认同样使用当前类的ClassLoader。你可以通过方法XX.class.GetClassLoader()获取。

 

Reference

[浅析Context Class Loader](http://blog.csdn.net/kabini/article/details/2975263)

[ContextClassLoader浅析](http://blog.csdn.net/qbg19881206/article/details/8890600)

<https://www.ibm.com/developerworks/cn/java/j-lo-classloader/>

<http://blog.sina.com.cn/s/blog_605f5b4f01010i48.html>

 <http://my.oschina.net/u/571166/blog/212903>

##org.apache.catalina.startup.Bootstrap's main method handles 'startd' or 'stopd' wrongly

ASF Bugzilla – Bug 47881	org.apache.catalina.startup.Bootstrap's main method handles 'startd' or 'stopd' wrongly	Last modified: 2009-11-02 16:47:52 UTC
Home | New | Browse | Search |  
  Search [?] | Reports | Help | New Account | Log In | Forgot Password
Bug 47881 - org.apache.catalina.startup.Bootstrap's main method handles 'startd' or 'stopd' wrongly
Status:	RESOLVED FIXED
Alias:	None
Product:	Tomcat 6
Component:	Catalina (show other bugs)
Version:	6.0.20
Hardware:	PC Linux
Importance:	P2 minor (vote)
Target Milestone:	default
Assignee:	Tomcat Developers Mailing List
URL:	
Keywords:	
Duplicates (1):	48059 (view as bug list)
Depends on:	
Blocks:	
 
Reported:	2009-09-20 19:14 UTC by qingyang.xu
Modified:	2009-11-02 16:47 UTC (History)
CC List:	0 users


Attachments
Add an attachment (proposed patch, testcase, etc.)

Note
You need to log in before you can comment on or make changes to this bug.
Description qingyang.xu 2009-09-20 19:14:04 UTC
String command = "start";
if (args.length > 0) {
    command = args[args.length - 1];
}

if (command.equals("startd")) {
    args[0] = "start";
    daemon.load(args);
    daemon.start();
} else if (command.equals("stopd")) {
    args[0] = "stop";
    daemon.stop();
} 
... ...


should be: 

String command = "start";
if (args.length > 0) {
    command = args[args.length - 1];
}
if (command.equals("startd")) {
    args[args.length - 1] = "start";
    daemon.load(args);
    daemon.start();
} else if (command.equals("stopd")) {
    args[args.length - 1] = "stop";
    daemon.stop();
} 
... ...

Please refer to the following usage method of  org.apache.catalina.startup.Catalina:
protected void usage() {

        System.out.println
            ("usage: java org.apache.catalina.startup.Catalina"
             + " [ -config {pathname} ]"
             + " [ -nonaming ] { start | stop }");

    }
Comment 1 Konstantin Kolinko 2009-10-08 05:31:24 UTC
Actually, I do not see startd and stopd commands to be documented anywhere. Maybe we can just safely remove them?

With startd, Tomcat will exit immediately upon startup. Though it may be useful for testing config files.

With stopd ... you cannot stop server that is not running.

And if you have an instance of Bootstrap, you can call load/start/stop directly, not relying on how Bootstrap#main() parses the arguments.

Regarding the patch: args are not used in "stopd", that line can be omitted completely, or we can do as OP proposes. Regarding "startd": yes, the fix is correct.
Comment 2 qingyang.xu 2009-10-08 16:20:38 UTC
(In reply to comment #1)
> Actually, I do not see startd and stopd commands to be documented anywhere.
> Maybe we can just safely remove them?
> 
> With startd, Tomcat will exit immediately upon startup. Though it may be useful
> for testing config files.
> 
> With stopd ... you cannot stop server that is not running.
> 
> And if you have an instance of Bootstrap, you can call load/start/stop
> directly, not relying on how Bootstrap#main() parses the arguments.
> 
> Regarding the patch: args are not used in "stopd", that line can be omitted
> completely, or we can do as OP proposes. Regarding "startd": yes, the fix is
> correct.


(In reply to comment #1)
> Actually, I do not see startd and stopd commands to be documented anywhere.
> Maybe we can just safely remove them?
> 
> With startd, Tomcat will exit immediately upon startup. Though it may be useful
> for testing config files.
> 
> With stopd ... you cannot stop server that is not running.
> 
> And if you have an instance of Bootstrap, you can call load/start/stop
> directly, not relying on how Bootstrap#main() parses the arguments.
> 
> Regarding the patch: args are not used in "stopd", that line can be omitted
> completely, or we can do as OP proposes. Regarding "startd": yes, the fix is
> correct.


The only difference between 'startd' and 'start' is that 'startd' doesn't invoke Catalina's await() method after the server has been started, or 'startd' doesn't need to allocate a 'SHUTDOWN' port to listening to be sent a 'SHUTDOWN' message. 'stopd' is the counterpart of 'stop'. If tomcat is started by 'startd', you must invoke 'stopd' to shutdown it, otherwise the server can't  be stopped, because 'stop' invoke Catalina's stopServer() method, which relies on the allocated 'SHUTDOWN' port exclusively. On the contrary, 'stopd' invokes Catalina's stop() method instead, which invoke server's stop() method directly. Below is the two different methods invoked by 'stop' and 'stopd':


//invoked by 'stop'
public void stopServer() {
        stopServer(null);
    }

    public void stopServer(String[] arguments) {

        if (arguments != null) {
            arguments(arguments);
        }

        if( server == null ) {
            // Create and execute our Digester
            Digester digester = createStopDigester();
            digester.setClassLoader(Thread.currentThread().getContextClassLoader());
            File file = configFile();
            try {
                InputSource is =
                    new InputSource("file://" + file.getAbsolutePath());
                FileInputStream fis = new FileInputStream(file);
                is.setByteStream(fis);
                digester.push(this);
                digester.parse(is);
                fis.close();
            } catch (Exception e) {
                log.error("Catalina.stop: ", e);
                System.exit(1);
            }
        }

        // Stop the existing server
        try {
            if (server.getPort()>0) {
            	String hostAddress = InetAddress.getByName("localhost").getHostAddress();
            	Socket socket = new Socket(hostAddress, server.getPort());
            	OutputStream stream = socket.getOutputStream();
            	String shutdown = server.getShutdown();
            	for (int i = 0; i < shutdown.length(); i++)
            		stream.write(shutdown.charAt(i));
            	stream.flush();
            	stream.close();
            	socket.close();
            } else {
                log.error(sm.getString("catalina.stopServer"));
                System.exit(1);
            }
        } catch (IOException e) {
            log.error("Catalina.stop: ", e);
            System.exit(1);
        }

    }


    // invoked by 'stopd'
    public void stop() {

        try {
            // Remove the ShutdownHook first so that server.stop() 
            // doesn't get invoked twice
            if (useShutdownHook) {
                Runtime.getRuntime().removeShutdownHook(shutdownHook);
            }
        } catch (Throwable t) {
            // This will fail on JDK 1.2. Ignoring, as Tomcat can run
            // fine without the shutdown hook.
        }

        // Shut down the server
        if (server instanceof Lifecycle) {
            try {
                ((Lifecycle) server).stop();
            } catch (LifecycleException e) {
                log.error("Catalina.stop", e);
            }
        }

    }


My suggestion: We don't need to remove 'startd' and 'stopd', they are useful in some situations. The only place we need to correct is the Bootstrap's main() method, as in the patches I have submitted. Thanks!
Comment 3 Konstantin Kolinko 2009-10-09 01:42:03 UTC
With "startd", as there is no "await", the method completes immediately (and the application quits).

That it is only useful if exiting that "main" method does not complete the application. Thus if it is not the actual entry point of the application. If so, does anyone need to use it that way?
Comment 4 qingyang.xu 2009-10-09 02:00:01 UTC
(In reply to comment #3)
> With "startd", as there is no "await", the method completes immediately (and
> the application quits).
> 
> That it is only useful if exiting that "main" method does not complete the
> application. Thus if it is not the actual entry point of the application. If
> so, does anyone need to use it that way?


Yes, you are totally right. After the main method ends, all the daemon threads (such as 'http-8080-Acceptor-0', 'TP-Processor', etc.) will end too. I never thought of this consequence. So, what's the point of 'startd' and 'stopd', anyway?
Comment 5 Filip Hanik 2009-10-09 06:48:53 UTC
(In reply to comment #4)
> (In reply to comment #3)
> > With "startd", as there is no "await", the method completes immediately (and
> > the application quits).
> > 
> > That it is only useful if exiting that "main" method does not complete the
> > application. Thus if it is not the actual entry point of the application. If
> > so, does anyone need to use it that way?
> 
> Yes, you are totally right. After the main method ends, all the daemon threads
> (such as 'http-8080-Acceptor-0', 'TP-Processor', etc.) will end too. I never
> thought of this consequence. So, what's the point of 'startd' and 'stopd',
> anyway?


####public static void main(String[] args) can be called from another Java class, its not exclusive to the JVM

It would be a different way of embedding Tomcat, at which point one wants the thread to return
Comment 6 qingyang.xu 2009-10-09 18:39:20 UTC
> public static void main(String[] args) can be called from another Java class,
> its not exclusive to the JVM
> 
> It would be a different way of embedding Tomcat, at which point one wants the
> thread to return


Understood. So my patch is still relevant, right?
Comment 7 qingyang.xu 2009-10-26 17:38:23 UTC

*** This bug has been marked as a duplicate of bug 48059 ***
Comment 8 Konstantin Kolinko 2009-11-02 02:45:38 UTC
Reduced severity. There is nothing in this issue that makes it "critical".
Comment 9 Konstantin Kolinko 2009-11-02 02:46:05 UTC
*** Bug 48059 has been marked as a duplicate of this bug. ***
Comment 10 Konstantin Kolinko 2009-11-02 03:04:19 UTC
Fixed in trunk, proposed for TC 6.0
Comment 11 Mark Thomas 2009-11-02 16:47:52 UTC
This has been applied to 6.0.x and will be included in 6.0.21 onwards.

