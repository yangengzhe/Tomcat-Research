## Servlet是如何工作的
1. 创建一个request对象，并填充信息。如：参数、头部、cookies、查询字符串等。**一个request对象是javax.servlet.ServletRequest或javax.servlet.http.ServletRequest接口的一个实列。**
2. 创建一个response对象，用于返回对用户的响应。**一个response对象是javax.servlet.ServletResponse或javax.servlet.http.ServletResponse接口的一个实列。**
3. 调用servlet的service方法，并传入request和response对象。

## Catalina 架构

Catalina 看成是由两个主要模 块所组成的:连接器(connector)和容器(container)。

- 连接器是用来“连接”容器里边的请求的。它的工作是为接收 到每一个 HTTP 请求构造一个 request 和 response 对象。然后它把流程传递给容器。
- 容器从连接 器接收到 requset 和 response 对象之后调用 servlet 的 service 方法用于响应。(其实还有很多，比如加载servlet，验证用户，更新会话等等)

## 预备知识 HTTP请求

HTTP请求是基于Socket实现的，是建立在TCP协议之上的一种应用，属于应用层的协议。我们可以用代码简单模拟下HTTP请求：

	Socket socket = new Socket("127.0.0.1", "8080");
	OutputStream os = socket.getOutputStream();
	boolean autoflush = true;
	PrintWriter out = new PrintWriter( socket.getOutputStream(), autoflush); 
	BufferedReader in = new BufferedReader(new InputStreamReader( socket.getInputstream() )); 
	//开始发送
	out.println("GET /index.jsp HTTP/1.1");
	out.println("Host: localhost:8080");
	out.println("Connection: Close");
	out.println();
	//读取响应
	boolean loop = true;
	StringBuffer sb = new StringBuffer(8096); 
	while (loop) {
		if ( in.ready() ) { 
			int i=0;
			while (i!=-1) {
				i = in.read();
				sb.append((char) i);
			}		loop = false;		}		Thread.currentThread().sleep(50);	}
	//收到的结果	System.out.println(sb.toString()); 	socket.close();

## 简单的Web服务器

一个简单的Web容器，我们认为是有三部分组成：HttpServer(看成连接器)、Request（看成容器）、Response（看成容器）

### HttpServer
实质就是SocketServer，目的是做HTTP请求服务器，用于接收HTTP请求，接收后创建Request和Response对象并交由程序继续执行。主要实现思路，创建一个ServerSocket，监听本地IP和端口，进行阻塞监听，直到有客户端发来请求后，封装Request和Response并处理。	ServerSocket serverSocket = null; 	int port = 8080;
	try {		serverSocket = new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1")); 
	}catch (IOException e) { 
		e.printStackTrace(); 
		System.exit(1);
	}
	// 等待请求的到来
	while (!shutdown) {
		Socket socket = null;
		InputStream input = null;
		OutputStream output = null; 
		try {
			socket = serverSocket.accept();			input = socket.getInputStream();			output = socket.getOutputStream();			//创建Request			Request request = new Request(input); 			request.parse();			//创建Response			Response response = new Response(output);			response.setRequest(request);			response.sendStaticResource();//发送数据
			//关闭Socket
			socket.close();
			//判断是否是关闭命令
			shutdown = request.getUri().equals(SHUTDOWN_COMMAND);        }        catch (Exception e) {			e.printStackTrace ();			continue; 		}	}
### Request
主要包含两个公共方法：parse和parseUri；两个属性：InputStream input 和 String uri
1.实例化Request时，初始化传入input；
2.parse()：用来读取input内容，并调用parseUri函数设置uri

例如：HTTP请求内容如下：

	GET /index.html HTTP/1.1
	
在parseUri中，通过获取第一个空格和第二个空格的方法，获取URI为 `/index.html` 并赋值给变量URI

### Response

主要包含两个公共方法：sendStaticResource 和 setRequest方法；两个属性：Request request和OutputStream output

1.实例化Response时，初始化传入output

2.sendStaticResource() 主要是读取资源文件（如FILE）并输出到output上