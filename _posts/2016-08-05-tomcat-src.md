
# 结构

```java
java-
	org.apache-
		catalina-
			startup-
				Bootstrap #1
```

## \#1 启动机制

**init class loaders（#1-1）- init org.apache.catalina.startup.Catalina (#1-2) - load catalinaDaemon (#1-3)**

### \#1-1 ClassLoader

|CLASS|NAME|PARENT|DEFAULT|
|:---:|:---:|:---:|:---:|
|commonLoader|commonLoader|NULL|JVM System Loader|
|catalinaLoader|server|commonLoader|commonLoader|
|sharedLoader|shared|commonLoader|commonLoader|

1. 读取catalina class loader配置加载classloader，查找配置文件顺序为：
	1. System.getProperty("catalina.config")
	2. new File("conf/catalina.properties")
	3. CatalinaProperties.class.getResourceAsStream("/org/apache/catalina/startup/catalina.properties")
2. Tomcat的currentThread设置为catalinaLoader

### \#1-2 init Catalina

1. 获取org.apache.catalina.startup.Catalina新实例
2. 初始化org.apache.catalina.startup.Catalina的ParentClassLoader为sharedLoader
3. catalinaDaemon指定为org.apache.catalina.startup.Catalina新实例

### \#1-3 load catalinaDaemon

#### invork catalinaDaemon的load方法

也就是org.apache.catalina.startup.Catalina新实例的load方法

**init tmpdir/naming/Digester (\#1-3-1) - load config file (\#1-3-2) - init catalina home/base path -init standard/error output stream - init server (\#1-3-3) - start**

1. \#1-3-1 init tmp dir, 读取java.io.tmpdir配置，创建临时目录，一般在/var下; Digester用于解析xml的工具
2. \#1-3-2 load config file，查找tomcat配置的顺序：
	1. getClassLoader().getResourceAsStream(getConfigFile()) //catalinaLoader, 传入的-config
	2. conf/server.xml //默认
	3. getClass().getClassLoader().getResourceAsStream("server-embed.xml") //在catalina.jar中
	使用Digester读取配置
3. \#1-3-3 初始化Server，在初始化过程中还初始化了Service-Connector
	![初始化结构](http://ww3.sinaimg.cn/mw690/a8484315jw1f6oo02184nj20f80djwey.jpg)
	
4. 启动过程中，
	1. fireLifecycleEvent(), 就是初始化tomcat中配置的Listener
	2. 

