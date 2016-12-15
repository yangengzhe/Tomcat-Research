## 安全组件

一个 servlet 通过一个叫 authenticator 的阀门(valve)来支持安全性限制。 当容器启动的时候,authenticator 被添加到容器的流水线上。authenticator 阀门会在包装器阀门之前被调用。authenticator 用于对用户进 行验证,如果用户熟人了正确的用户名和密码,authenticator 阀门调用下一个 用于处理请求 servlet 的阀门。如果验证失败,authenticator 不唤醒下一个阀 门直接返回。由于验证失败,用户并不能看到请求的 servlet。

本文主要介绍跟安全性相关的类(realms、principal、roles)

## Realm（域）

域是用于进行用户验证的一个组件,一个上下文只可以关联一个域，通过域可以验证用户名和密码是否是合法的。一个域所拥有的用户名和密码是可以访问他们的（例如Tomcat的默认合法用户存放在tomcat-users.xml文件里）但是可以使用域的其它 实现来访问其它的源,如关系数据库。

	public Principal authenticate(String username, String credentials);	public Principal authenticate(String username, byte[] credentials);	public Principal authenticate(String username, String digest, String nonce, String nc, String cnonce, String qop, String realm, String md5a2);	public Principal authenticate(X509Certificate certs[]);
	
	public boolean hasRole(Principal principal, String role);## GenericPrincipal
即上文提到的java.security.Principal接口的实现类——GenericPrincipal。一个GenericPrincipal必须跟一个域相关联。
	public GenericPrincipal(Realm realm, String name, String password) {		this(realm, name, password, null);
	}
	public GenericPrincipal(Realm realm, String name, String password, List roles) {    	super();    	this.realm = realm;		this.name = name;    	this.password = password;    	if (roles != null) {      		this.roles = new String[roles.size()];      		this.roles = (String[]) roles.toArray(this.roles);      		if (this.roles.length > 0)        		Arrays.sort(this.roles);    	}	}

## LoginConfig

LoginConfig 的实例封装了 域名和验证要用的方法。可以使用 LoginConfig 实例的 getRealmName 方法来获 得域名,可以使用 getAuthName 方法来验证用户。

**Tomcat 一个部署启动的时候,先读取 web.xml。如果 web.xml 包括一个 login-config 元素,Tomcat 创建一个 LoginConfig 对象并相应的设置它的属性。 验证域名调用 LoginConfig 的 getRealmName 方法并将域名发送给浏览器显示登录表单。如果 getRealmName 名字返回值为 null,则发送给浏览器服务器的名字 和端口名。**

## Authenticator

org.apache.catalina.Authenticator 接口用来表示一个验证器。该方接口并没 有方法,只是一个组件的标志器,这样就能使用 instanceof 来检查一个组件是 否为验证器。