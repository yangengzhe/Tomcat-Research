## Serlvet容器
## 第一章
1. 等待HTTP请求。
2. 构造一个ServletRequest对象和一个ServletResponse对象。
3. 假如该请求需要一个静态资源的话，调用StaticResourceProcessor实例的process方法，同时传递ServletRequest和ServletResponse对象。
4. 假如该请求需要一个servlet的话，加载servlet类并调用servlet的service方法，同时传递ServletRequest和ServletResponse对象。

##本章——Servlet的原理实现 Request和Response

Servlet编程是通过javax.servlet和javax.servlet.http这两个包的类和接口来实现的。其中一个至关重要的就是javax.servlet.Servlet接口了。所有的servlet必须实现实现或者继承实现该接口的类。

> 在Servlet的五个方法中，init，service和destroy是servlet的生命周期方法.

INIT:Servlet初始化后只调用一次init方法，可以通过覆盖这个方法来写那些仅仅只要运行一次的初始化代码，例如加载数据库驱动，值初始化等等。在其他情况下，这个方法通常是留空的。

SERVICE:servlet容器为servlet请求调用它的service方法。service方法将会给调用多次。

DESTORY:当从服务中移除一个servlet实例的时候，servlet容器调用destroy方法。这通常发生在servlet容器正在被关闭或者servlet容器需要一些空闲内存的时候

###整体流程:（生命周期）
1、当第一次调用servlet的时候，加载该servlet类并调用servlet的init方法(仅仅一次)。
2、对每次请求，构造一个javax.servlet.ServletRequest实例和一个javax.servlet.ServletResponse实例。
3、调用servlet的service方法，同时传递ServletRequest和ServletResponse对象。
4、当servlet类被关闭的时候，调用servlet的destroy方法并卸载servlet类。

## servlet容器 HttpServer1

- 对于每一个HTTP请求，servlet容器必须构造一个ServletRequest对象和一个ServletResponse对象并把它们传递给正在服务的servlet的service方法。
- 用来生成资源库的代码是从org.apache.catalina.startup.ClassLoaderFactory的createClassLoader方法来的，而生成URL的代码是从org.apache.catalina.loader.StandardClassLoader的addRepository方法来的。

注：为了提高安全性，保证Servlet底层代码的安全有两种办法：1、设置可见性为包内可见；2、利用外观模式Facade进行封装。（推荐） 增加RequestFacade 和 ResponseFacade两个类，分别对原有Request类和Response类进行封装