## Tomcat的默认连接器

> 以前一节的基础，可以更好的理解本章所讲的内容——tomcat4的默认连接器。

Tomcat连接器是一个可以插入servlet容器的独立模块，之所以称之为模块，显然代码是解耦合的，只需要实现接口符合一定标准，可以插入servlet容器中即可。因此目前市面上有很多连接器，本例中以tomcat4默认连接器为例，一个Tomcat连接器必须符合：

1. 必须实现接口org.apache.catalina.Connector
2. 必须创建请求对象,该请求对象的类必须实现接口 org.apache.catalina.Request
3. 必须创建响应对象,该响应对象的类必须实现接口 org.apache.catalina.Response

### 工作原理

等待前来的 HTTP 请求,创建 request 和 response 对象,然后把 request 和 response 对象传递给容器。

连接器是通过调用接口 org.apache.catalina.Container 的 invoke 方法来传递 request 和 response 对象的。
	
	void invoke(org.apache.catalina.Request request,org.apache.catalina.Response response);在 invoke 方法里边,容器加载 servlet,调用它的 service 方法,管理会话,记录出错日志等等。
### 准备工作——org.apache.catalina.Connector 接口
Tomcat连接器必须实现org.apache.catalina.Connector 接口，在该接口的众多方法中最重要的是 getContainer,setContainer, createRequest 和 createResponse
- setContainer 是用来关联连接器和容器用的
- getContainer 返回关联的容器
- createRequest 为前来的 HTTP 请求构造一个请求对象
- createResponse 创建一个响应对象

类 org.apache.catalina.connector.http.HttpConnector 是 Connector 接口的一个实现,将在下面进行讨论

### HttpConnector类（线程）

连接器的入口，处理最底层socket。负责创建服务器套接字，并接受客户端的链接，并维护，最终传递给HttpProcessor去解析、处理套接字（即HTTP请求）

主要完成的功能是：

- 创建一个服务器套接字。initialize方法调用私有open方法，利用工厂返回ServerSocket实例。
- 维护 HttpProcessor 实例。因为一个HttpProcessor实例只能处理一个HTTP请求，维护线程池，可以同时处理多个请求。
- 为 HTTP 请求服务。将客户socket传递给HttpProcessor。调用processor.assign(socket)方法

### HttpProcessor类（线程）

用线程池进行管理，每一个HttpProcessor对应一个Socket链接（即一个HTTP请求），对socket进行解析并封装成request和response传给容器进一步使用。

重要方法：

- assign 用于接收、存储socket，并利用wait/notify通知await
- await 利用wait/notify等待通知，一旦打断则返回socket，传递socket运行process
- process 用于解析HTTP请求和调用容器的invoke方法

**如何实现异步？！**

	HttpConnector							HttpProcessor
	
	1.接收客户端socket
	2.HttpProcessor.assign(socket)--------->1.调用assign方法
												存储socket
		直接返回结果（实现异步）	<----------------┘   |(notifyAll打断全部wait)
	3.返回1循环执行								    2.唤醒await
												      执行process处理HTTP请求
												      
主要完成的功能：

- 创建请求对象（HttpRequestImpl HttpRequestFacade等）
- 创建响应对象（HttpResponseImpl HttpResponseFacade等）
- 处理HTTP请求，即process方法：解析链接（parseConnection 方法从套接字中获取到网络地址并把它赋予 HttpRequestImpl 对象 包含：IP、端口等）、解析请求（parseRequest）、解析头部（parseHeaders）

### 简单容器 => 模拟tomcat

运行流程：

Bootstrap：负责启动应用Tomcat（也就是启动连接器container）
Connector：连接器，负责打开socket接收HTTP，解析HTTP，封装request和response并交给container容器处理
Container：Servlet容器，实现 org.apache.catalina.container 接口，主要完成了invoke方法

#### SimpleContainer类

核心方法：

	public void invoke(Request request, Response response) throws IoException, ServletException {
		string servletName = ( (Httpservletrequest) request).getRequestURI();
		servletName = servletName.substring(servletName.lastIndexof("/") + 1);
		URLClassLoader loader = null;
		try {
			URL[] urls = new URL[1];
			URLStreamHandler streamHandler = null;
			File classpath = new File(WEB_ROOT);
			string repository = (new URL("file",null,classpath.getCanonicalpath() + File.separator)).toString();
			urls[0] = new URL(null, repository, streamHandler);
			loader = new URLClassLoader(urls);
		} catch (IOException e) {
			System.out.println(e.toString() );
		}
		
		Class myClass = null;
		try {
			myClass = loader.loadclass(servletName);
		}catch (classNotFoundException e) {
			System.out.println(e.toString());
		}
		
		servlet servlet = null;
		try {
			servlet = (Servlet) myClass.newInstance();
			servlet.service((HttpServletRequest) request, (HttpServletResponse) response);
		} catch (Exception e) {
			System.out.println(e.toString());
		} catch (Throwable e) {
			System.out.println(e.toString());
		}
	}
	
