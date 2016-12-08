## 连接器

> 本章开始进行分模块介绍，这里主要介绍三个模块connector、startup和core

startup模块：只有一个类Bootstrap，用来启动应用的。

connector（连接器）模块：主要分为五部分
1）连接器和它的支撑类（HttpConnector和HttpProcessor）
2）指代HTTP请求的类（HttpRequest）和它的辅助类
3）指代HTTP响应的类（HttpResponse）和它的辅助类
4）Facade类（HttpRequestFacade 和 HttpResponseFacade）
5）Constant类

core模块（类似于Servlet容器）：由两个类组成 ServletProcessor和StaticResourceProcessor类。负责处理servlet请求

## 连接器的实现

Tomcat 的默认连接器和我们的连接器使用 SocketInputStream 类来从套接字的 InputStream 中读取字节流。

### 启动应用程序——Bootstrap 类

	public final class Bootstrap {		public static void main(String[] args) { 
			HttpConnector connector = new 	HttpConnector(); 
			connector.start();	} }

Bootstrap类主要作用是启动connector线程。

### 连接器

#### HttpConnector类（线程）

HttpConnector是连接器类，是一个线程继承Runnable接口。主要任务是创建服务器套接字，并接受客户端的连接。

主要完成的功能是：

- 监听8080端口，启动Socket，并阻塞等待客户端的连接（等待HTTP请求）
- 当收到请求后，为每个请求创建HttpProcessor实例
- 调用HttpProcessor的process方法，传递socket

#### HttpProcessor（这节中暂不考虑成线程）

用于接收HttpConnector传来的客户端socket。一个实例处理一个HTTP请求。

主要完成的功能是：

- 创建一个HttpRequest对象，将InputStream封装成HttpRequest
- 创建一个HttpResponse对象，将OnputStream封装成HttpResponse
- 解析HTTP请求的第一行和头部，并放到HttpRequest对象
- 将HttpRequest和HttpResponse对象发送给到一个ServletProcessor或者StaticResourceProcessor（传递给Servlet容器 进行下一步内容）。

#### HttpRequest

实现了javax.servlet.http.HttpServletRequest接口，跟随它的是一个叫做 HttpRequestFacade 的 facade 类。

主要完成的功能是：

- 读取套接字输入流 InputStream
- 解析请求行（调用readRequestLine方法） =>HttpProcessor.parseRequest. 完成设置setQueryString、setRequestedSessionId等
- 解析头部（调用SocketInputStream.readHeader方法，获得头部HttpHeader类）=>HttpProcessor.parseHeaders
- 解析cookies(调用 org.apache.catalina.util.RequestUtil 的 parseCookieHeader 方法)
- 获取参数。不需要马上解析查询字符串或者 HTTP 请求内容,直到 servlet 需要通过调用 javax.servlet.http.HttpServletRequest 的 getParameter,getParameterMap, getParameterNames 或者 getParameterValues 方法来读取参数。

#### HttpResponse

实现了 javax.servlet.http.HttpServletResponse。跟随它的是一个叫做 HttpResponseFacade 的 facade 类。

主要完成的功能是：

- 输出内容（getWriter方法） 利用OutputStream完成数据的发送

#### Facade

主要是处于安全的对HttpRequest和HttpResponse的封装，详细看上一节

#### Constant

系统相关信息的存储，比如目录等配置文件

### core模块

#### Servlet容器（静态资源处理器和 Servlet 处理器）

主要是上一节内容，不做更多的变化

## 总结

在本章中,你已经知道了连接器是如何工作的。建立起来的连接器是 Tomcat4 的默认连接器 的简化版本。正如你所知道的,因为默认连接器并不高效,所以已经被弃用了。例如,所有的 HTTP 请求头部都被解析了,即使它们没有在 servlet 中使用过。因此,默认连接器很慢,并且 已经被 Coyote 所代替了。Coyote 是一个更快的连接器,它的源代码可以在 Apache 软件基金会 的网站中下载。不管怎样,默认连接器作为一个优秀的学习工具,将会在下一章中详细讨论。