## 容器

> 容器是一个处理用户 servlet 请求并返回对象给 web 用户的模块。

org.apache.catalina.Container 接口定义了容器的形式,一个容器必须实现该接口。Tomcat主要有四种容器:**Engine (引擎), Host(主机), Context(上下文), 和 Wrapper(包装器)**

- Engine:表示整个 Catalina 的 servlet 引擎(标准实现StandardEngine)
- Host:表示一个拥有数个上下文的虚拟主机（标准实现StandardHost）
- Context:表示一个 Web 应用,一个 context 包含一个或多个 wrapper（标准实现StandardContext）
- Wrapper:表示一个独立的 servlet（标准实现StandardWrapper）

不一定非要包含全部的4种容器，一个容器可以有一个或多个低层次上的子容器。如第一节中只有一个wrapper容器；第二节中有一个Context容器和两个wrapper容器。而 wrapper 作为容器层次中的最底层,不能包含子容器。

本文主要介绍Context和Wrapper容器

### Pipelining Tasks(流水线任务)

主要是org.apache.catalina 中四个相关的接口:Pipeline, Valve, ValveContext

一个流水线就像一个过滤链,每一个阀门像一个过滤器。跟过滤器一样,一个阀门可以操作传递给它的 request 和 response 方法。让一个阀门完成了处理,则进一步处理流水线中的下一个阀门,**基本阀门总是在最后才被调用。**

为了便于理解，用以下伪代码：

	//按顺序执行流水线的每一个阀门
	for (int n=0; n<valves.length; n++) {
		valve[n].invoke( ... );//调用invoke就认为处理了阀门	}
	//最后执行基础阀门
	basicValve.invoke( ... );

#### Pipeline接口

唤醒流水线的阀门；增加或删除一个阀门；配置基本阀门；最后执行基本阀门。（基本阀门负责处理request和回复response）

	public interface Pipeline {
		public Valve getBasic();
		public void setBasic(Valve valve);//设置基本类
		public void addValve(Valve valve);//添加阀门
		public Valve[] getValves();//删除阀门
		public void invoke(Request request, Response response) throws IOException, ServletException;//执行入口
		public void removeValve(Valve valve);
	}

#### ValveContext（内部类）阀门上下文接口

Pipeline类的内部类，用于保证添加给它的阀门必须被调用一次

主要成员：

- List<Valve> 用来存储每一个阀门

主要方法：

- invokeNext(Request request,Response response)

接口内容：

	public interface ValveContext {
		public String getInfo();
		public void invokeNext(Request request, Response response) throws IOException, ServletException;
	}

StandardValveContext类的部分实现：

	public void invokeNext(Request request, Response response) throws IOException, ServletException {
		int subscript = stage;
		stage = stage + 1;
		//执行下一个valves
		if (subscript < valves.length) {
			valves[subscript].invoke(request, response, this);
		}else if ((subscript == valves.length) && (basic != null)) {
			//如果都执行完了，执行基础valve      		basic.invoke(request, response, this);      	}else {      		throw new ServletException(sm.getString("standardPipeline.noValve"));      	}    }

#### Valve阀门接口

主要方法：

- invoke(Request request, Response response,ValveContext valveContext)

接口内容： 

	public interface Valve {
		public String getInfo();
		public void invoke(Request request, Response response, ValveContext context) throws IOException, ServletException;
	}

部分实现：

	public void invoke(Request request, Response response, ValveContext valveContext) throws IOException, ServletException {
		//调用下一个阀门
		valveContext.invokeNext(request, response);
		//当前阀门做的事情
		...
	}

### 容器Contained接口

#### Wrapper 接口（包装器）

一个包装器是表示一个 独立 servlet 定义的容器。包装器继承了 Container 接口,并且添加了几个方法。 包装器的实现类负责管理其下层 servlet 的生命中期,包括 servlet 的 init,service,和 destroy 方法。由于包装器是最底层的容器,所以不可以将子 容器添加给它。如果 addChild 方法被调用的时候会产生 IllegalArgumantException 异常。

#### 上下文(Context)接口

一个 context 在容器中表示一个 web 应用。一个 context 通常含有一个或多个包 装器作为其子容器。

重要的方法包括 addWrapper, createWrapper 等方法。包含属性：ContextMapper（用于管理子容器，但是在tomcat5以上就不再用了）和 ContextValve基本阀门

Mapper的接口：

	public interface Mapper {
		public Container getContainer();
		public void setContainer(Container container);		public String getProtocol();		public void setProtocol(String protocol);		public Container map(Request request, boolean update);	}

## 总结

### 单Wrapper容器

#### 准备工作

SimpleLoader，用于获取类加载器（ClassLoader）进行加载Servlet

	ClassLoader classLoader;//类加载器
	Container container;//容器

SimplePipeline，实现invoke方法，包含内部类 SimplePipelineValveContext。用于控制管道和阀门。

	class SimplePipelineValveContext{}
	
	public void setBasic(Valve valve);//设置基本类
	public Valve[] getValves();//获得阀门
	public void invoke(Request request, Response response) throws IOException, ServletException;//执行入口
		
	Valve[] valves;
	int stage;//当前要执行的阀门

SimpleWrapper，实现了 allocate 方法和 load 方法；包含loader 变量用于加载一个 servlet 类。和Parent 变量表示该包装器的父容器。（例如可以是Context的子容器）该类有一个流水线和该流水线的基本阀门
	
	allocate()//获得servlet对象
	load()
	public SimpleWrapper() {		pipeline.setBasic(new SimpleWrapperValve());	}
	
	SimplePipeline pipeline;//管道	Loader loader;
	Container parent = null;//父容器	
SimpleWrapperValve类是一个给 SimpleWrapper 类专门处理请求的**基本阀门**。最后必须执行的阀门

	public void invoke(Request request, Response response, ValveContext valveContext) throws IOException, ServletException {
		SimpleWrapper wrapper = (SimpleWrapper) getContainer();
		ServletRequest sreq = request.getRequest();
		ServletResponse sres = response.getResponse();
		Servlet servlet = null;
		HttpServletRequest hreq = null;
		if (sreq instanceof HttpServletRequest)
			hreq = (HttpServletRequest) sreq;
		HttpServletResponse hres = null;
		if (sres instanceof HttpServletResponse)
		    hres = (HttpServletResponse) sres;
		try {
			servlet = wrapper.allocate();//获取Servlet实例
			if (hres!=null && hreq!=null) {
				servlet.service(hreq, hres);
			}else {      			servlet.service(sreq, sres);      		}      	}catch (ServletException e) {      	}    }
多个Valve：ClientIPLoggerValve、HeaderLoggerValve
Bootstrap1：启动应用

	public static void main(String args[]){
		HttpConnector connector = new HttpConnector();
		Wrapper wrapper = new SimpleWrapper();    	wrapper.setServletClass("ModernServlet");//包装执行的Servlet    	Loader loader = new SimpleLoader();    	Valve valve1 = new HeaderLoggerValve();    	Valve valve2 = new ClientIPLoggerValve();    	wrapper.setLoader(loader);    	((Pipeline) wrapper).addValve(valve1);    	((Pipeline) wrapper).addValve(valve2);    	connector.setContainer(wrapper);    	try {      		connector.initialize();      		connector.start();      		// 暂停      		System.in.read();    	}catch (Exception e) {      		e.printStackTrace();      	}
	}

#### 流程

1.启动连接器connector监听客户端。

2.创建Wrapper容器，并包装Servlet（一个Wrapper容器包装一个Servlet），指指定名字不加载和实例化

3.创建类加载器

4.创建阀门

5.为Wrapper容器添加阀门和加载器

6.为连接器管理Wrapper容器

===

7.启动连接器，收到请求解析后通过Container（容器）调用Pipeline的invoke方法
	
	public void invoke(Request request, Response response) throws IOException, ServletException {
		pipeline.invoke(request, response);
	}

8.Pipeline（流水线）invoke按照顺序调用valve阀门的invoke方法（ClientIPLoggerValve、HeaderLoggerValve）

9.保证顺序是通过传递参数PipelineValveContext（内部类）的invokeNext方法，按照数组逐渐调用

10.最后调用基础Valve类（SimpleWrapperValve）。通过wrapper.allocate();获取Servlet实例

11.执行Servlet的方法，servlet.service(hreq, hres);

### 单Context容器，多Wrapper子容器 

Context容器中，当有多于一个得包装器的时候,需要一个 mapper 来处理这些子容器, 对于特殊的请求可以使用特殊的子容器来处理。（一个容器也 可以有多个 mapper 来支持多协议。例如容器可以用一个 mapper 来支持 HTTP 协 议,而使用另一个 mapper 来支持 HTTPS 协议。）
#### 流程

**单容器**

	Wrapper容器 -> pipeline(Context) -> valve -> ... -> BaseValve -> Servlet -> servlet.service(request,response)
	
**多容器**

	Context容器 -> pipeline(Context) -> valve -> ... -> BaseValve -> 用Mapper查找负责的子容器 —> Wrapper容器 -> ...同上

1. Context容器有一个流水线,容器的 invoke 方法会调用流水线的 invoke 方法。2. 流水线的 invoke 方法会调用添加到容器中的阀门的 invoke 方法, 然后调用基本阀门的 invoke 方法。3. （单容器）在一个包装器中,基本阀门负责加载相关的 servlet 类并对请求作 出相应。4. （多容器）在一个有子容器的上下文中,基本法门使用 mapper 来查找负责处 理请求的子容器。如果一个子容器被找到,子容器的 invoke 方法会被调 用,然后返回步骤 1。