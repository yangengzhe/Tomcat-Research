## 加载器

> 使用JAVA的类加载器在加载Servlet使可以进入CLASSPATH目录，加载任何类。处于对安全性的考虑，Servlet只允许加载WEB-INF目录下的类以及WEB-INF/lib目录下的类库。所以servlet容器需要一个自己的容器。

Tomcat 需要一个自己的加载器的另一个原因是它需要支持在 WEB-INF/classes 或者是 WEB-INF/lib 目录被改变的时候会重新加载。

要实现一个Tomcat的类加载器必须实现org.apache.catalina.Loader 接口

Tomcat 的加载器实现中使用一个单独的线程来检查 servlet 和该Servlet类文件的时间戳。要完成Servlet类自动加载功能,一个加载器类必须实现 org.apache.catalina.loader.Reloader 接口。

### JAVA类加载器

每次在创建一个类实例时，前提是必须将该类已经加载到内存中了。而这个将类从硬盘的.class文件加载到内存中的过程，就是用JVM的类加载器来进行加载的。

Java 加载器在 Java 核心类库和 CLASSPATH 环境下面的所有类中查找类。如果需要的类找不到,会抛出 java.lang.ClassNotFoundException 异常。

JVM的类加载器采用双亲委派形式，主要有三种加载器：Bootstrap 类加载器、extension 类加载器和 systen 类加载器。

- bootstrap 类加载器用于引导 JVM,一旦调用 java.exe 程序,bootstrap 类加载器就开始工作。因此,它必须使用本地代码实现,如C语言等，然后加载 JVM 需要的类到函数中。另外,它还负责加载所有的Java核心类,例如java.lang和java.io 包。另外 bootstrap 类加载器还会查找核心类库如 rt.jar、i18n.jar 等,这些 类库根据 JVM 和操作系统来查找。
- extension 类加载器负责加载标准扩展目录下面的类。这样就可以使得编写程序变得简单,只需把JAR文件拷贝到扩展目录下面即可,类加载器会自动的在下面查找。不同的供应商提供的扩展类库是不同的,Sun 公司的 JVM 的标准扩展目录是/jdk/jre/lib/ext。
- system 加载器是默认的加载器,它在环境变量 CLASSPATH 目录下面查找相应的类。

类加载的顺序是利用委派模型，自顶向下的加载，这样的好处是出于安全的考虑。比如一个程序员恶意写了一个java.lang.Object 类，若不采用委派模型，直接加载该程序员自定义的java.lang.Object 类，会导致系统瘫痪，出现安全隐患。但是一旦采用了该模式，流程将变为：

	system 类加载器将该请求委派给 extension 类加载器
	extension 类加载器委派给 bootstrap 类加载器
	bootstrap 类加载器先搜索的核心库,找到标准 java.lang.Object 加载并实例化它
	因此，自定义的 java.lang.Object 类永远不会被加载
	
### Loader 接口

	public interface Loader {		public ClassLoader getClassLoader();		public Container getContainer();
		public void setContainer(Container container);		public DefaultContext getDefaultContext();		public void setDefaultContext(DefaultContext defaultContext);    	public boolean getDelegate();     	public void setDelegate(boolean delegate);    	public String getInfo();    	public boolean getReloadable();    	public void setReloadable(boolean reloadable);    	public void addPropertyChangeListener(PropertyChangeListener    listener);    	public void addRepository(String repository);    	public String[] findRepositories();    	public boolean modified();    	public void removePropertyChangeListener(PropertyChangeListener listener);    }
### Reloader 接口
	public interface Reloader {    	public void addRepository(String repository);    	public String[] findRepositories ();    	public boolean modified();    }
## 实现类加载器——WebappLoader类
> org.apache.catalina.loader.WebappLoader 类是 Loader 接口的实现,它表示 一个 web 应用程序的加载器,负责给 web 应用程序加载类。
主要完成的功能：
- 创建一个类加载器- 设置库
- 设置类路径
- 设置访问权限
- 开启一个新线程用来进行自动重载
- 缓存类列表
- 加载类

### 创建一个类加载器

	public class WebappLoader implement Loader,Reloader{
		WebappLoader classLoader;
		String loaderClass = "org.apahce.catalina.loader.WebappClassLoader";
		
		String getLoaderClass();
		void setLoaderClass(String loaderClass);
		WebappLoader createClassLoader();
	}

WebappLoader实现Loader接口，含有一个ClassLoader类型的类加载器成员变量。当WebappLoader被初始化时，并没有构造函数和setClassLoader方法用来初始化加载器，那么getClassLoader方法返回的ClassLoader实例是怎么来的呢？

WebappLoader 类提供了 getLoaderClass 和 setLoaderClass 方法来获得或者改变它的私有变量 loaderClass 的值。该变量是一个的表示加载器类名 String 类型表示形式。默认的 loaderClass 值是 org.apahce.catalina.loader.WebappClassLoader,当 WebappLoader 启动的时候,它会使用它的私有方法 createClassLoader 创建 WebappClassLoader 的实例

	private WebappClassLoader createClassLoader() throws Exception {
		Class clazz = Class.forName(loaderClass);
		WebappClassLoader classLoader = null;
		if (parentClassLoader == null) {
			classLoader = (WebappClassLoader) clazz.newInstance();
		}else {    		Class[] argTypes = { ClassLoader.class };    		Object[] args = { parentClassLoader };    		Constructor constr = clazz.getConstructor(argTypes);    		classLoader = (WebappClassLoader) constr.newInstance(args);		}		return classLoader;	}

### 设置库

WebappLoader 的 start 方法会调用 setRepositories 方法来给类加载器添加一个库。比如：WEB-INF/classes 目录传递给加载器 addRepository 方法,而 WEB-INF/lib 传递给加载器的 setJarPath 方法。

### 设置类路径

该任务由 start 方法调用 setClassPath 方法完成,setClassPath 方法会给 servlet 上下文分配一个 String 类型属性保存 Jasper JSP 编译的类路径

### 设置访问权限

如果 Tomcat 使用了安全管理器,setPermissions 给类加载器给必要的目录添加 访问权限,例如 WEB-INF/classes 和 WEB-INF/lib。如果不使用管理器,该方法 马上返回。

### 开启自动重载线程

WebappLoader 虽然集成了Reloader类，但是仍然不支持自动重载，为了实现这个目的,WebappLoader实现了 java.lang.Runnable 从而有一个单独的线程每个 x 秒(checkInterval)会检查源的时间戳。

	public void run() {
		while (!threadDone) {    		threadSleep();//等待checkInterval秒    		if (!started)
				break; 
			try {      			if (!classLoader.modified())//如果没有修改
					continue;
			}catch (Exception e) {      			continue;    		}
			//说明被修改了
			notifyContext();//调用重载方法
			break;
		}
	}
	
	//利用线程，进行重载
	private void notifyContext() {
		WebappContextNotifier notifier = new WebappContextNotifier();
		(new Thread(notifier)).start();
	}
	//实际的重载方法
	protected class WebappContextNotifier implements Runnable {
		public void run() {
			((Context) container).reload();
		}
	}

### 缓存

为了提高性能,当一个类被加载的时候会被放到缓存中,这样下次需要加载该类 的时候直接从缓存中调用即可。

每一个可以被加载的类(放在 WEB-INF/classes 目录下的类文件或者 JAR 文件) 都被当做一个源。一个源被 org.apache.catalina.loader.ResourceEntry 类表示，一个 ResourceEntry 实例保存一个 byte 类型的数组表示该类、最后修改的 数据或者副等等。

	public class ResourceEntry {
		public long lastModifled = -1;
		public byte[] binaryContent = null;
		public Class loadedClass = null;
		public URL source = null;
		public URL CodeBase = null;
		public Manifest manifest = null;
		public Certificate[] certificates = null;
	}

**所有缓存的源被存放在一个叫做 resourceEntries 的 HashMap 中,键值为源名, 所有找不到的源都被放在一个名为 notFoundResources 的 HashMap 中。**

### 加载类

当加载一个类的时候,WebappClassLoader 类遵循以下规则（流程）：

- （ 1.缓存 ）所有加载过的类都要进行缓存,所以首先需要检查本地缓存。
- （ 2.更高级缓存 ）如果无法在本地缓存找到类,使用 java.langClassLoader 类的 findLoaderClass 方法在缓存查找类
- （ 3.系统类加载器加载 ）如果在两个缓存中都无法找到该类,使用系统的类加载器避免从 J2EE 类中覆盖来的 web 应用程序。
- （ 4.检查是否允许加载 ）如果使用了安全管理器,检查该类是否允许加载,如果该类不 允许加载,则抛出 ClassNotFoundException 异常。
- （ 5-1.利用委派模式加载 ）如果要加载的类使用了委派标志或者该类属于 trigger 包中, 使用父加载器来加载类,如果父加载器为 null,使用系统加载器加载。
- （ 5-2-1.利用非委派模式加载，直接加载 ）从当前的源中加载类
- （ 5-2-2.利用非委派模式加载，自己无法加载 则变成委派形式 ）如果在当前的源中找不到该类并且没有使用委派标志,使用父 类加载器。如果父类加载器为 null,使用系统加载器
- （ 5-2-3.利用非委派模式加载，父类也无法加载，则报错 ）如果该类仍然找不到,抛出 ClassNotFoundException 异常

