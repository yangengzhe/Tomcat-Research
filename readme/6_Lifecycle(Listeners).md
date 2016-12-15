## 生命周期

> 当Tomcat启动的时候，要保证所有的组件都启动；当Tomcat停止的时候，要保证这些组件也必须有机会被停止。

一个容器可以包含一系列的组件如加载器、管理器等。一个父组件负责启动和停止其子组件。所有启动Tomcat只需要启动一个父组件即可，而不用逐个启动每个组件。

### Lifecycle 接口

	public interface Lifecycle {
		public static final String BEFORE_STOP_EVENT = "before_stop";
		public static final String AFTER_STOP_EVENT = "after_stop";
		
		public void addLifecycleListener(LifecycleListener listener);
		public LifecycleListener[] findLifecycleListeners();
		public void removeLifecycleListener(LifecycleListener listener);
		public void start() throws LifecycleException;
		public void stop() throws LifecycleException;
	}

### LifecycleEvent 类

表示生命周期中某一刻发生的某一事件

	Lifecycle lifecycle;
	String type;
	Object data;

### LifecycleListener 接口

表示生命周期监器

	public interface LifecycleListener {




		public LifecycleSupport(Lifecycle lifecycle);
		public void addLifecycleListener(LifecycleListener listener);		public LifecycleListener[] findLifecycleListeners();
		public void fireLifecycleEvent(String type, Object data);//执行事件
		public void removeLifecycleListener(LifecycleListener listener);


	
	LifecycleSupport lifecycle;//用于管理生命周期
	
			// 启动管道    		
			if (pipeline instanceof Lifecycle)
				((Lifecycle) pipeline).start();
			//通知所有START_EVENT的事件
		//通知所有AFTER_START_EVENT的事件
		lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);
	}


	Wrapper容器（addLifecycleListener添加监听器） -> fireLifecycleEvent(BEFORE_START_EVENT) -> pipeline(Context) -> valve -> ... -> BaseValve -> Servlet -> servlet.service(request,response) -> fireLifecycleEvent(START_EVENT) -> fireLifecycleEvent(AFTER_START_EVENT)
	
**多容器**

	Context容器 -> fireLifecycleEvent(BEFORE_START_EVENT) -> pipeline(Context) -> valve -> ... -> BaseValve -> 用Mapper查找负责的子容器 —> Wrapper容器 -> ...同上 .. -> fireLifecycleEvent(START_EVENT) -> fireLifecycleEvent(AFTER_START_EVENT)