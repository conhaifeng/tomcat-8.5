# Tomcat

Tomcat是运行在Apache之上的应用服务器，为客户端提供可以调用的方法。Tomcat是一个Servlet容器（可以认为Apache的扩展），可独立运行。



## 基本结构

![](README.assets/5d0af9e5e4b039f39f3f4bca.png)

## 源码解析

> 原版Maven Tomcat源码来自简书[《maven构建tomcat 源码》](https://www.jianshu.com/p/27124c44191d)

$$
Tomcat简单类图
$$

![1570099760627](README.assets/1570099760627.png)

| 类/接口            | 作用                                                         |
| ------------------ | ------------------------------------------------------------ |
| Bootstrap          | 启动类，封装了Catalina的基本方法（通过反射）                 |
| CatalinaProperties | 初始化Tomcat的配置并注册到System中                           |
| Catalina           | 启动/关闭命令行程序                                          |
| Digester           | 配置解析器，封装了XMLReader，通过配置可读取XML并转化成对象   |
| Lifecycle          | 生命周期接口，拥有init、start、stop、destroy四个周期API以及添加生命周期监听器LiftcycleListener的API<br />继承自此接口的类有：Server、Service、Connector、Engine、Host、Context |
| Server             | 代表Tomcat服务器                                             |
| Service            | 代表一个服务                                                 |
| Connector          | 代表一个连接器，负责维护客户端和服务器端的连接               |
| Engine             | 代表                                                         |
| Host               | 代表一个虚拟主机，当浏览器访问                               |
| Context            | 代表一个Web应用                                              |



### Bootstrap启动类

Bootstrap拥有main方法，是Tomcat的启动类
$$
Main方法——Bootstrap.main
$$

```java
/**
* 初始化daemon并使用daemon执行命令（无参数默认启动，有参数则根据参数启动/停止/配置测试）
* @param args 命令参数
*/
public static void main(String args[]) {

    //初始化daemon
    if (daemon == null) {
        // 创建一个Bootstrap对象，初始化并赋值给daemon
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.init();
        }
        //简单打印错误堆栈，然后重新抛出
        catch (Throwable t) {
            handleThrowable(t);
            t.printStackTrace();
            return;
        }
        daemon = bootstrap;
    } else {
        //有另一个线程已经为daemon赋值，则让当前线程使用daemon的上下文类加载器
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }

    //使用daemon执行命令,默认start(调用Catalina的setAwait、load、start)
    try {
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
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
            if (null == daemon.getServer()) {
                System.exit(1);
            }
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null == daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting
        if (t instanceof InvocationTargetException &&
            t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }

}
```

$$
初始化启动器init
$$

```java
public void init() throws Exception {

    //初始化类加载器
    initClassLoaders();

    //为当前线程设置上下文类加载器
    Thread.currentThread().setContextClassLoader(catalinaLoader);
    SecurityClassLoad.securityClassLoad(catalinaLoader);

    
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    //加载启动类——Catalina，并获取其实例
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    //反射调用Catalina的setParentClassLoader方法设置父类加载器为sharedLoader
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    //将实例化出的Catalina设置为Catalina守护
    catalinaDaemon = startupInstance;

}
```

$$
初始化类加载器initClassLoader
$$

```java
private void initClassLoaders() {
    try {
    	//调用createClassLoader创建common加载器
        commonLoader = createClassLoader("common", null);
        if( commonLoader == null ) {
            // no config file, default to this loader - we might be in a 'single' env.
            commonLoader=this.getClass().getClassLoader();
        }
        //调用createClassLoader创建catalina加载器和shared加载器
        catalinaLoader = createClassLoader("server", commonLoader);
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}
```

$$
创建类加载器createClassLoader
$$

```java
//创建出的类加载器是加载指定路径jar包的UrlClassLoader
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {
    //根据加载器名字从Catalina属性集获取属性值（来自catalina.properties）
    //Debug：value="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
    String value = CatalinaProperties.getProperty(name + ".loader");
    if ((value == null) || (value.equals("")))
        return parent;

    value = replace(value);

    List<Repository> repositories = new ArrayList<>();

    String[] repositoryPaths = getPaths(value);

    //根据类加载器属性（jar包地址）生成资源集合
    for (String repository : repositoryPaths) {
        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(
                new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(
                new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(
                new Repository(repository, RepositoryType.DIR));
        }
    }

    //根据jar包路径资源集合创建类加载器——加载指定路径下jar包的UrlClassLoader
    return ClassLoaderFactory.createClassLoader(repositories, parent);
}
```



### CatalinaProperties配置

CatalinaProperties在加载期获取Catalina的配置并注册到System属性中
$$
加载Catalina配置
$$

```java
static {
    loadProperties();
}

//加载属性集并注册到System中
private static void loadProperties() {

    //属性文件输入流
    InputStream is = null;
    try {
        //属性文件路径：系统属性"catalina.config"
        String configUrl = System.getProperty("catalina.config");
        if (configUrl != null) {
            is = (new URL(configUrl)).openStream();
        }
    } catch (Throwable t) {
        handleThrowable(t);
    }

    //系统属性为空则 Tomcat项目根目录\conf\catalina.properties文件
    if (is == null) {
        try {
            File home = new File(Bootstrap.getCatalinaBase());
            File conf = new File(home, "conf");
            File propsFile = new File(conf, "catalina.properties");
            is = new FileInputStream(propsFile);
        } catch (Throwable t) {
            handleThrowable(t);
        }
    }

    //Tomcat项目根目录\conf\catalina.properties文件不存在则class目录/org/apache/catalina/startup/catalina.properties文件
    if (is == null) {
        try {
            is = CatalinaProperties.class.getResourceAsStream
                ("/org/apache/catalina/startup/catalina.properties");
        } catch (Throwable t) {
            handleThrowable(t);
        }
    }

    //配置文件不为空，加载配置
    if (is != null) {
        try {
            properties = new Properties();
            properties.load(is);
        } catch (Throwable t) {
            handleThrowable(t);
            log.warn(t);
        } finally {
            try {
                is.close();
            } catch (IOException ioe) {
                log.warn("Could not close catalina.properties", ioe);
            }
        }
    }

    //属性文件仍旧为空，使用空值对象取代null（新建Properties对象）
    if ((is == null)) {
        // Do something
        log.warn("Failed to load catalina.properties");
        // That's fine - we have reasonable defaults.
        properties = new Properties();
    }

    // 最后把属性全部注册进System中
    Enumeration<?> enumeration = properties.propertyNames();
    while (enumeration.hasMoreElements()) {
        String name = (String) enumeration.nextElement();
        String value = properties.getProperty(name);
        if (value != null) {
            System.setProperty(name, value);
        }
    }
}
```

$$
配置文件catalina.properties
$$

```properties
#配置访问权限(UrlClassLoader在加载类时会核对权限)，拥有RuntimePermission ("accessClassInPackage."+package)权限方可访问
package.access=sun.,org.apache.catalina.,org.apache.coyote.,org.apache.jasper.,org.apache.tomcat.

#配置能够通过checkPackageDefinition的权限：RuntimePermission ("defineClassInPackage."+package)
package.definition=sun.,java.,org.apache.catalina.,org.apache.coyote.,\
org.apache.jasper.,org.apache.naming.,org.apache.tomcat.

#配置类加载器的加载路径（用于加载Catalina的类加载器）
common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"

#配置类加载器的加载路径（用于加载Server）
server.loader=

#共享型类加载器的加载路径
shared.loader=

# 配置JarScanner跳过扫描的jar包
# The JARs listed below include:
# - Tomcat Bootstrap JARs
# - Tomcat API JARs
# - Catalina JARs
# - Jasper JARs
# - Tomcat JARs
# - Common non-Tomcat JARs
# - Test JARs (JUnit, Cobertura and dependencies)
tomcat.util.scan.StandardJarScanFilter.jarsToSkip=\
annotations-api.jar,\
ant-junit*.jar,\
ant-launcher.jar,\
ant.jar,\
asm-*.jar,\
aspectj*.jar,\
bootstrap.jar,\
catalina-ant.jar,\
catalina-ha.jar,\
catalina-jmx-remote.jar,\
catalina-storeconfig.jar,\
catalina-tribes.jar,\
catalina-ws.jar,\
catalina.jar,\
cglib-*.jar,\
cobertura-*.jar,\
commons-beanutils*.jar,\
commons-codec*.jar,\
commons-collections*.jar,\
commons-daemon.jar,\
commons-dbcp*.jar,\
commons-digester*.jar,\
commons-fileupload*.jar,\
commons-httpclient*.jar,\
commons-io*.jar,\
commons-lang*.jar,\
commons-logging*.jar,\
commons-math*.jar,\
commons-pool*.jar,\
dom4j-*.jar,\
easymock-*.jar,\
ecj-*.jar,\
el-api.jar,\
geronimo-spec-jaxrpc*.jar,\
h2*.jar,\
hamcrest-*.jar,\
hibernate*.jar,\
httpclient*.jar,\
icu4j-*.jar,\
jasper-el.jar,\
jasper.jar,\
jaspic-api.jar,\
jaxb-*.jar,\
jaxen-*.jar,\
jdom-*.jar,\
jetty-*.jar,\
jmx-tools.jar,\
jmx.jar,\
jsp-api.jar,\
jstl.jar,\
jta*.jar,\
junit-*.jar,\
junit.jar,\
log4j*.jar,\
mail*.jar,\
objenesis-*.jar,\
oraclepki.jar,\
oro-*.jar,\
servlet-api-*.jar,\
servlet-api.jar,\
slf4j*.jar,\
taglibs-standard-spec-*.jar,\
tagsoup-*.jar,\
tomcat-api.jar,\
tomcat-coyote.jar,\
tomcat-dbcp.jar,\
tomcat-i18n-*.jar,\
tomcat-jdbc.jar,\
tomcat-jni.jar,\
tomcat-juli-adapters.jar,\
tomcat-juli.jar,\
tomcat-util-scan.jar,\
tomcat-util.jar,\
tomcat-websocket.jar,\
tools.jar,\
websocket-api.jar,\
wsdl4j*.jar,\
xercesImpl.jar,\
xml-apis.jar,\
xmlParserAPIs-*.jar,\
xmlParserAPIs.jar,\
xom-*.jar

# 需要扫描的jar包（jarsToSkip配置的目录下可能有需要扫描的jar包）
tomcat.util.scan.StandardJarScanFilter.jarsToScan=\
log4j-taglib*.jar,\
log4j-web*.jar,\
log4javascript*.jar,\
slf4j-taglib*.jar

# 字符串缓存配置
tomcat.util.buf.StringCache.byte.enabled=true
#tomcat.util.buf.StringCache.char.enabled=true
#tomcat.util.buf.StringCache.trainThreshold=500000
#tomcat.util.buf.StringCache.cacheSize=5000
```



### Catalina

Catalina是真正启动Tomcat的类（Bootstrap相当于Catalina的对外代理人）
$$
构造函数
$$

```java
public Catalina() {
    //向Security注册访问权限
    setSecurityProtection();
    //空方法，提前触发ExceptionUtils的加载
    ExceptionUtils.preload();
}
protected void setSecurityProtection(){
    //获取SecurityConfig单例，SecurityConfig从catalina.properties文件中读取Security属性（没有则使用默认值）并设置到Security中
    SecurityConfig securityConfig = SecurityConfig.newInstance();
    //设置包定义权限
    securityConfig.setPackageDefinition();
    //设置包访问权权限
    securityConfig.setPackageAccess();
}
```

$$
加载load
$$

```java

```

$$
启动start
$$

```java

```



### 配置解析器Digester

 封装了XMLReader，通过配置可读取XML并转化成对象，用于将服务器配置（默认server.xml）转化为Server对象
$$
被配置——Catalina的createStartDigester()
$$

```java
protected Digester createStartDigester() {
    long t1=System.currentTimeMillis();
    // Initialize the digester
    Digester digester = new Digester();
    digester.setValidating(false);
    digester.setRulesValidation(true);
    Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
    List<String> objectAttrs = new ArrayList<>();
    objectAttrs.add("className");
    fakeAttributes.put(Object.class, objectAttrs);
    // Ignore attribute added by Eclipse for its internal tracking
    List<String> contextAttrs = new ArrayList<>();
    contextAttrs.add("source");
    fakeAttributes.put(StandardContext.class, contextAttrs);
    digester.setFakeAttributes(fakeAttributes);
    digester.setUseContextClassLoader(true);

    // Configure the actions we will be using
    digester.addObjectCreate("Server",
                             "org.apache.catalina.core.StandardServer",
                             "className");
    digester.addSetProperties("Server");
    digester.addSetNext("Server",
                        "setServer",
                        "org.apache.catalina.Server");

    digester.addObjectCreate("Server/GlobalNamingResources",
                             "org.apache.catalina.deploy.NamingResourcesImpl");
    digester.addSetProperties("Server/GlobalNamingResources");
    digester.addSetNext("Server/GlobalNamingResources",
                        "setGlobalNamingResources",
                        "org.apache.catalina.deploy.NamingResourcesImpl");

    digester.addObjectCreate("Server/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Listener");
    digester.addSetNext("Server/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service",
                             "org.apache.catalina.core.StandardService",
                             "className");
    digester.addSetProperties("Server/Service");
    digester.addSetNext("Server/Service",
                        "addService",
                        "org.apache.catalina.Service");

    digester.addObjectCreate("Server/Service/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Service/Listener");
    digester.addSetNext("Server/Service/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    //Executor
    digester.addObjectCreate("Server/Service/Executor",
                             "org.apache.catalina.core.StandardThreadExecutor",
                             "className");
    digester.addSetProperties("Server/Service/Executor");

    digester.addSetNext("Server/Service/Executor",
                        "addExecutor",
                        "org.apache.catalina.Executor");


    digester.addRule("Server/Service/Connector",
                     new ConnectorCreateRule());
    digester.addRule("Server/Service/Connector",
                     new SetAllPropertiesRule(new String[]{"executor", "sslImplementationName"}));
    digester.addSetNext("Server/Service/Connector",
                        "addConnector",
                        "org.apache.catalina.connector.Connector");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig",
                             "org.apache.tomcat.util.net.SSLHostConfig");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig",
                        "addSslHostConfig",
                        "org.apache.tomcat.util.net.SSLHostConfig");

    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                     new CertificateCreateRule());
    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                     new SetAllPropertiesRule(new String[]{"type"}));
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/Certificate",
                        "addCertificate",
                        "org.apache.tomcat.util.net.SSLHostConfigCertificate");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                             "org.apache.tomcat.util.net.openssl.OpenSSLConf");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                        "setOpenSslConf",
                        "org.apache.tomcat.util.net.openssl.OpenSSLConf");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                             "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                        "addCmd",
                        "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");

    digester.addObjectCreate("Server/Service/Connector/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Service/Connector/Listener");
    digester.addSetNext("Server/Service/Connector/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service/Connector/UpgradeProtocol",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Service/Connector/UpgradeProtocol");
    digester.addSetNext("Server/Service/Connector/UpgradeProtocol",
                        "addUpgradeProtocol",
                        "org.apache.coyote.UpgradeProtocol");

    // Add RuleSets for nested elements
    digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
    digester.addRuleSet(new EngineRuleSet("Server/Service/"));
    digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
    digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
    addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
    digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));

    // When the 'engine' is found, set the parentClassLoader.
    digester.addRule("Server/Service/Engine",
                     new SetParentClassLoaderRule(parentClassLoader));
    addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");

    long t2=System.currentTimeMillis();
    if (log.isDebugEnabled()) {
        log.debug("Digester for server.xml created " + ( t2-t1 ));
    }
    return (digester);

}
```

$$
解析服务器配置文件生成Server对象
$$

```java
public Object parse(InputSource input) throws IOException, SAXException {
    //确认是否配置过
    configure();
    //用自身配置配置XMLReader，然后用XMLReader解析XML文件
    getXMLReader().parse(input);
    return (root);
}
```

$$
配置文件server.xml
$$

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- Server -->
<Server port="8005" shutdown="SHUTDOWN">

    <!--几个生命周期监听器-->
    <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

    <!--全局JNDI服务命名资源集-->
    <GlobalNamingResources>

        <!--用户数据库（xml数据库），用于让UserDatabaseRealm校验用户身份-->
        <Resource name="UserDatabase" auth="Container"
                  type="org.apache.catalina.UserDatabase"
                  description="User database that can be updated and saved"
                  factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                  pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>

    <!-- Service -->
    <Service name="Catalina">

        <!--Connector的共享执行器-->
        <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
                  maxThreads="150" minSpareThreads="4"/>


        <!--Connector连接器，拥有执行器、端口、协议、超时时限和重定向端口等属性-->
        <Connector executor="tomcatThreadPool"
                   port="8080" protocol="HTTP/1.1"
                   connectionTimeout="20000"
                   redirectPort="8443" />


        <!--定义一个 实现NIO的基于SSL/TLS HTTP/1.1连接的连接器-->
        <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
                   maxThreads="150" SSLEnabled="true">
            <SSLHostConfig>
                <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                             type="RSA" />
            </SSLHostConfig>
        </Connector>


        <!-- 定义一个 协议为 AJP 1.3 的连接器 -->
        <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />


        <!-- Engine：处理请求，分析请求头然后传递给指定的主机Host -->
        <Engine name="Catalina" defaultHost="localhost">


            <!-- 用户身份权限认证 -->
            <Realm className="org.apache.catalina.realm.LockOutRealm">
                <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                       resourceName="UserDatabase"/>
            </Realm>

            <!--虚拟主机-->
            <Host name="localhost"  appBase="webapps"
                  unpackWARs="true" autoDeploy="true">

                <!--单例值，保存用户认证信息，此值在多个Web应用下共享-->
                <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                       prefix="localhost_access_log" suffix=".txt"
                       pattern="%h %l %u %t &quot;%r&quot; %s %b" />

            </Host>
        </Engine>
    </Service>
</Server>
```



### 生命周期Lifecycle

Lifecycle接口代表组件的生命周期，拥有init、start、stop、destory四个生命周期方法，并可添加LifecycleListener生命周期监听器，并定义了若干监听器触发事件（初始化前    初始化后    启动前    启动    启动后    停止前    停止    停止后    销毁前    销毁后）

#### 骨架类LifecycleBase

简单实现Lifecycle接口——同步生命周期API，确保生命周期状态正常变更，并将具体实现职责下放到XXXInternal中
$$
构造函数
$$

```java
//未声明构造函数，默认构造函数
```

$$
init
$$

```java
public final synchronized void init() throws LifecycleException {
    //状态为New方可初始化
    if (!state.equals(LifecycleState.NEW)) {
        //抛出状态异常
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        //状态变更——初始化中
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        //初始化具体逻辑
        initInternal();
        //状态变更——初始化后
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(
            sm.getString("lifecycleBase.initFail",toString()), t);
    }
}
```

$$
start
$$

```java
//状态为New——现初始化
//状态为Failed——停止
//状态不为初始化完成或已停止——生命周期异常
public final synchronized void start() throws LifecycleException {
    //状态为启动前、中、后——不可启动
    if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
        LifecycleState.STARTED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
        } else if (log.isInfoEnabled()) {
            log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
        }

        return;
    }

    //状态为New——现初始化
    if (state.equals(LifecycleState.NEW)) {
        init();
        //状态为Failed——停止
    } else if (state.equals(LifecycleState.FAILED)) {
        stop();
        //状态不为初始化完成或已停止——生命周期异常
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
               !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    try {
        //状态变更——启动前
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        startInternal();
        //状态为启动失败——停止
        if (state.equals(LifecycleState.FAILED)) {
            stop();
            //状态不为启动中——生命周期异常
        } else if (!state.equals(LifecycleState.STARTING)) {
            invalidTransition(Lifecycle.AFTER_START_EVENT);
            //状态变更——启动后
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(sm.getString("lifecycleBase.startFail", toString()), t);
    }
}
```

$$
stop
$$

```java
//New——直接设置为停止后
//启动后/启动失败——执行停止
public final synchronized void stop() throws LifecycleException {

    //状态为停止前、中、后——生命周期异常
    if (LifecycleState.STOPPING_PREP.equals(state) || LifecycleState.STOPPING.equals(state) ||
        LifecycleState.STOPPED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyStopped", toString()), e);
        } else if (log.isInfoEnabled()) {
            log.info(sm.getString("lifecycleBase.alreadyStopped", toString()));
        }

        return;
    }
    //状态为New——变更为已停止
    if (state.equals(LifecycleState.NEW)) {
        state = LifecycleState.STOPPED;
        return;
    }

    //状态不为启动后或启动失败——生命周期异常
    if (!state.equals(LifecycleState.STARTED) && !state.equals(LifecycleState.FAILED)) {
        invalidTransition(Lifecycle.BEFORE_STOP_EVENT);
    }

    try {
        if (state.equals(LifecycleState.FAILED)) {
            //触发监听器事件——停止前
            fireLifecycleEvent(BEFORE_STOP_EVENT, null);
        } else {
            setStateInternal(LifecycleSta  te.STOPPING_PREP, null, false);
        }

        stopInternal();

        if (!state.equals(LifecycleState.STOPPING) && !state.equals(LifecycleState.FAILED)) {
            invalidTransition(Lifecycle.AFTER_STOP_EVENT);
        }

        setStateInternal(LifecycleState.STOPPED, null, false);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(sm.getString("lifecycleBase.stopFail",toString()), t);
    } finally {
        if (this instanceof Lifecycle.SingleUse) {
            // Complete stop process first
            setStateInternal(LifecycleState.STOPPED, null, false);
            destroy();
        }
    }
}
```

$$
destory
$$

```java
//启动失败——先调停止，再触发销毁
//New、初始化后、启动后、停止后——触发销毁
public final synchronized void destroy() throws LifecycleException {
    //启动失败——触发停止
    if (LifecycleState.FAILED.equals(state)) {
        try {
            // Triggers clean-up
            stop();
        } catch (LifecycleException e) {
            // Just log. Still want to destroy.
            log.error(sm.getString("lifecycleBase.destroyStopFail", toString()), e);
        }
    }

    //销毁中或已销毁——生命周期异常
    if (LifecycleState.DESTROYING.equals(state) ||
        LifecycleState.DESTROYED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyDestroyed", toString()), e);
        } else if (log.isInfoEnabled() && !(this instanceof Lifecycle.SingleUse)) {
            // Rather than have every component that might need to call
            // destroy() check for SingleUse, don't log an info message if
            // multiple calls are made to destroy()
            log.info(sm.getString("lifecycleBase.alreadyDestroyed", toString()));
        }

        return;
    }

    //状态不为已停止或启动失败或New或初始化完成——生命周期异常
    if (!state.equals(LifecycleState.STOPPED) &&
        !state.equals(LifecycleState.FAILED) &&
        !state.equals(LifecycleState.NEW) &&
        !state.equals(LifecycleState.INITIALIZED)) {
        invalidTransition(Lifecycle.BEFORE_DESTROY_EVENT);
    }

    try {
        setStateInternal(LifecycleState.DESTROYING, null, false);
        destroyInternal();
        setStateInternal(LifecycleState.DESTROYED, null, false);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(
            sm.getString("lifecycleBase.destroyFail",toString()), t);
    }
}
```



#### LifecycleMBeanBase

Tomcat构件基础，Tomcat的构件基本都继承或间接继承自此类
$$
构造函数
$$

```java
//未声明构造函数，默认构造函数
```

$$
init
$$

```java

```

$$
start
$$

```java

```

$$
stop
$$

```java

```

$$
destory
$$

```java

```



### 组件注册器Registry

组件（MBean）注册器：提供创建和操作组件以及简化组件使用的类，其本身也是一个组件

实现了RegistryMBean接口和MBeanRegistration接口

#### RegistryMBean接口

组件注册器的主要接口：提供创建和操作组件以及简化组件使用的方法

| 作用                                       | API                          |
| ------------------------------------------ | ---------------------------- |
| 注册/取消注册组件                          | register/unRegisterComponent |
| 调用指定组件列表中所有组件的一个指定的操作 | invoke                       |
|                                            | getId                        |
|                                            | stop                         |



#### MBeanRegistration接口

组件注册的监听器接口，提供注册、取消注册触发事件的方法

| 作用              | API                |
| ----------------- | ------------------ |
| 注册前/后触发     | pre/postRegister   |
| 取消注册前/后触发 | pre/postDeregister |



### 服务器Server

Server代表整个Servlet容器

Server接口继承自Lifecycle接口，并提供了大量服务器相关API
$$
API
$$

| 类型      | 作用                     | API                         |
| --------- | ------------------------ | --------------------------- |
| 获取/设置 | 所在Catalina             | get/setCatalina             |
|           | Catalina的base路径       | get/setCatalinaBase         |
|           | Catalina的home路径       | get/setCatalinaHome         |
|           | 全局命名资源             | get/setGlobalNamingResource |
|           | 全局命名上下文           | get/setGlobalNamingContext  |
|           | 等待的Shutdown命令字符串 | get/setShutdown             |
|           | 地址                     | get/setAddress              |
|           | 端口号                   | get/setPort                 |
| 其他      | 等待直到Shutdown命令到达 | await                       |
|           | 查找Service              | findService(String name)    |
|           | 获取Service集合          | findService                 |
|           | 增删Service              | add/removeService           |



#### 实现类StandardServer

继承自LifecycleMBeanBase类，实现Server接口
$$
构造函数
$$

```java

```

$$
初始化实现——initInternal
$$

```java
protected void initInternal() throws LifecycleException {

    super.initInternal();

    // JNDI服务端注册字符串缓存服务（类型为StringCache）
    onameStringCache = register(new StringCache(), "type=StringCache");

    // JNDI服务端注册组件工程服务（类型为MBeanFactory）
    MBeanFactory factory = new MBeanFactory();
    factory.setContainer(this);
    onameMBeanFactory = register(factory, "type=MBeanFactory");

    // 从JDNI命名资源集中提取数据注册到JNDI服务
    globalNamingResources.init();

    // 加载类（来自Catalina.properties，在Bootstrap的initClassLoader方法中将jar路径存放至Url类加载器并将此类加载器存放至Catalina中）
    if (getCatalina() != null) {
        ClassLoader cl = getCatalina().getParentClassLoader();

        while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
            if (cl instanceof URLClassLoader) {
                URL[] urls = ((URLClassLoader) cl).getURLs();
                for (URL url : urls) {
                    if (url.getProtocol().equals("file")) {
                        try {
                            File f = new File (url.toURI());
                            if (f.isFile() &&
                                f.getName().endsWith(".jar")) {
                                ExtensionValidator.addSystemResource(f);
                            }
                        } catch (URISyntaxException e) {
                            // Ignore
                        } catch (IOException e) {
                            // Ignore
                        }
                    }
                }
            }
            cl = cl.getParent();
        }
    }
    // 初始化所有Service
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### Service

Service是一个包含多个连接器Connector和一个主机容器Engine的容器，以便于：多个连接器统一为同一个主机容器提供请求-响应连接。

Service接口继承自LifeCycle接口，额外提供Service的API。

| 类别      | 作用             | API                      |
| --------- | ---------------- | ------------------------ |
| 获取/设置 | 所在服务器Server | get/setServer            |
|           | 名字             | get/setServer            |
|           | 类加载器         | get/setParentClassLoader |
|           | 主机容器Engine   | get/setContainer         |
| 增/删/查  | 连接器Connector  | add/remove/findConnector |
|           | 连接器共享执行器 | add/remove/findExecutor  |



#### StandardService

Service的默认实现类，继承自LifeCycleMBean
$$
构造函数
$$

```java

```


$$
初始化实现——initInternal
$$

```java

```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### 连接器Connector

$$
构造函数
$$

```java

```

$$
初始化实现——initInternal
$$

```java

```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### 主机容器Engine

Engine是包含多个主机的容器，它封装多个主机，以便：

- 统一拦截到达他们的请求；
- 让他们统一使用同一个连接器

Engine实现LifeCycle接口，额外提供Engine的API。

| 作用                 | API                |
| -------------------- | ------------------ |
| 获取/设置所在Service | get/setService     |
| 获取/设置默认主机    | get/setDefaultHost |
| 获取/设置JVM路由ID   | get/setJvmRoute    |



#### StandardEngine

Engine的默认实现类，继承自ContainerBase（间接继承自LifecycleMBeanBase）
$$
构造函数
$$

```java

```

$$
初始化实现——initInternal
$$

```java
protected void initInternal() throws LifecycleException {
    // 确保权限认证Realm存在
    getRealm();
    super.initInternal();
}
public Realm getRealm() {
    Realm configured = super.getRealm();
    //权限认证Realm不存在——返回空值对象
    if (configured == null) {
        configured = new NullRealm();
        this.setRealm(configured);
    }
    return configured;
}

```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### 虚拟主机Host

$$
构造函数
$$

```java

```

$$
初始化实现——initInternal
$$

```java

```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### Web应用Context

$$
构造函数
$$

```java

```

$$
初始化实现——initInternal
$$

```java

```

$$
启动实现——startInternal
$$

```java

```

$$
停止实现——stopInternal
$$

```java

```

$$
销毁实现——destoryInternal
$$

```java

```



### Servlet映射器Mapper





