---
layout: post
title:  "java之SPI机制"
categories: jdk dubbo
tags:  spi dubbo
author: zhangxiankui
---

* content
{:toc}


## 一、SPI简介
所谓SPI，就是Service Provider Interface的简称，字面意思是service服务接口，其实我理解就是为了使框架的可扩展性更好，并且在扩展时不需要改动jar包，
只需要自定义就可以，如dubbo中扩展服务过滤器、服务拦截器等

## 二、JAVA SPI
#### 1.java spi约定
- 1）在META-INF/services/目录中创建以接口全限定名命名的文件，该文件内容为Api具体实现类的全限定名
- 2）ServiceLoader类去加载该目录下的实现类
- 3）实现类必须包含无参构造放方法

#### 2.java spi demo
##### api以及实现类
- 1）api
```
public interface SpiDemoService {
    void sayHello(String content);
}
```
- 2）实现类

```
public class DoctorService implements SpiDemoService {

    @Override
    public void sayHello(String content) {
        System.out.println("Hello " + content + ", I'm Doctor!");
    }
}

public class TeacherService implements SpiDemoService {

    @Override
    public void sayHello(String content) {
        System.out.println("Hello " + content + ", I'm Teacher!");
    }
}
```
##### META-INF/services/com.zxk.service.SpiDemoService文件配置
```
com.zxk.service.impl.jdk.TeacherService
com.zxk.service.impl.jdk.DoctorService
```

##### JdkRun
```
public class JdkRun {

    public static void main(String[] args) {
        ServiceLoader<SpiDemoService> services = ServiceLoader.load(SpiDemoService.class);

        services.forEach(spiDemoService -> {
            spiDemoService.sayHello("world!");
        });
    }

}
```
运行结果：
```
Hello world!, I'm Teacher!
Hello world!, I'm Doctor!
```
#### 3.java spi 实现原理
- 核心实现类就是ServiceLoader类，结合嵌套Iterator特性来实现懒加载，具体可以自行翻阅JDK源码
- 使用迭代器实现的好处是方便迭代具体的实现类
- 使用内部类LazyIterator来实现懒加载，这很好的封装了查找的过程，这个实现很值得借鉴

#### 4.java spi的应用-DriverManger

##### 1)DriverManager连接mysql的demo
```java
// 1.注册驱动
Class.forName("com.mysql.jdbc.Driver");

// 2.获取连接
Connection connection =
        DriverManager.getConnection("jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8", "root", "");

// 3.获取Statement
String sql = "select * from user where id=?";
PreparedStatement preparedStatement = connection.prepareStatement(sql);
preparedStatement.setInt(1, 1);

// 4.获取查询的结果集
ResultSet resultSet = preparedStatement.executeQuery();
if (resultSet.next()) {
    System.out.println("name:" + resultSet.getString("name") + ", age:"
            + resultSet.getInt("age"));
```
我们知道如果想要通过DriverManager来连接oracle，上面代码只需要将驱动和对应的url更改就可以了，那么DriverManager到底是如何实现多个数据驱动管理的呢？没错，就是java的spi

##### 2）DriverManager源码之加载多数据库驱动
```java
// 静态加载Drivers
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}

private static void loadInitialDrivers() {
    ...
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {

						// 这里是不是出现了我们熟悉的ServiceLoader，也就是说会加载java.sql.Driver文件
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();

            /* Load these drivers, so that they can be instantiated.
             * It may be the case that the driver class may not be there
             * i.e. there may be a packaged driver with the service class
             * as implementation of java.sql.Driver but the actual class
             * may be missing. In that case a java.util.ServiceConfigurationError
             * will be thrown at runtime by the VM trying to locate
             * and load the service.
             *
             * Adding a try catch block to catch those runtime errors
             * if driver not available in classpath but it's
             * packaged as service and that service is there in classpath.
             */
            try{
            		// 初始化所有加载到的数据库驱动
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });
    ...
}

private static Connection getConnection(
  String url, java.util.Properties info, Class<?> caller) throws SQLException {
  ...
  // 可以看出DriverManager通过遍历所有的驱动来找到适配的Driver，并获取数据库连接，这里其实有优化的空间，可以调用acceptUrl方法来过滤Driver
  for(DriverInfo aDriver : registeredDrivers) {
      // If the caller does not have permission to load the driver then
      // skip it.
      if(isDriverAllowed(aDriver.driver, callerCL)) {
          try {
          
          	/**
             * 其实java.sql.Driver接口定义了一个方法判断驱动能不能接受url的连接，
             *   boolean acceptsURL(String url) throws SQLException;
             *   这里其实先使下面代码进行判断

             *   if (!aDriver.driver.acceptsURL(url)) {
             *       continue;
             *  }

             *  但是没有使用上面代码进行判断，我猜可能的原因是：
             *  1、java官方并不要求实现acceptsURL()方法，我们平常 
             *  使用也很早用到这个方法。
             *  2、存在一种这样的情况：某个数据库厂商协议进行升级
             * 了，但是为了兼容旧的协议还是允许连接，
             * acceptsURL(旧协议的url)方法，返回false，
             * 起到一个提示的作用。
                
             */
              println("    trying " + aDriver.driver.getClass().getName());
              Connection con = aDriver.driver.connect(url, info);
              if (con != null) {
                  // Success!
                  println("getConnection returning " + aDriver.driver.getClass().getName());
                  return (con);
              }
          } catch (SQLException ex) {
              if (reason == null) {
                  reason = ex;
              }
          }

      } else {
          println("    skipping: " + aDriver.getClass().getName());
      }

  }
	...
}
```
从上面的加载中可看出，mysql或者oracle的驱动jar包中必然会存在java.sql.Driver文件，大家可以去对应的驱动包下去找下，下面以mysql为例来看下
![](https://zhangxiankui.github.io/imgs/spi/mysqlDriver.jpg)
```
static {
    try {
    		// 注册驱动到DriverManager的registerDrivers中
        java.sql.DriverManager.registerDriver(new Driver());
    } catch (SQLException E) {
        throw new RuntimeException("Can't register driver!");
    }
}
```
也就是说驱动静态注册到了DriverManger中了，这样其实就有了双重保障，一个是静态注册，一个是spi，那么手动调用代码Class.forName("")是使用静态注册，不调用就会使用spi


#### 5.java spi的缺点
- 必须一次加载所有接口的实现类来遍历，不能通过某个唯一标识符来单独获取某个实现类
- 这个类不是单例的


## 三.DUBBO SPI使用以及实现原理
#### 1.Dubbo SPI的demo
##### 1）service&impl
```
// 这个注解必须要加
@SPI
public interface SpiDemoService {
    void sayHello(String content);
}

public class DubboProfessor implements SpiDemoService {

    @Override
    public void sayHello(String content) {
        System.out.println("Hello " + content + ", I'm Professor!");
    }
}

public class DubboTechnogy implements SpiDemoService {

    @Override
    public void sayHello(String content) {
        System.out.println("Hello " + content + ", I'm Technogy!");
    }
}
```

##### 2）service配置
```
technogy=com.zxk.service.impl.dubbo.DubboTechnogy
professor=com.zxk.service.impl.dubbo.DubboProfessor
```

##### 3) 通过dubbo的spi机制来获取service
```
public static void main(String[] args) {
    ExtensionLoader<SpiDemoService> extensionLoader =
            ExtensionLoader.getExtensionLoader(SpiDemoService.class);
    SpiDemoService technogy = extensionLoader.getExtension("technogy");
    technogy.sayHello("world");
    SpiDemoService professor = extensionLoader.getExtension("professor");
    professor.sayHello("world");
}

#运行结果
Hello world, I'm Technogy!
Hello world, I'm Professor!
```
可以看出dubbo spi可以通过对应的key值来寻找对应的实现类，这也奠定了dubbo能够实现自定义扩展的特性，这是dubbo非常重要的一个特性，接下来看一下ExtensionLoader如何实现

#### 2.ExtensionLoader的源码分析
##### 1）getExtension
```
public T getExtension(String name) {
    ...
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 获取extension
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

private T createExtension(String name) {
    // 获取class
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 注入对应的属性
        injectExtension(instance);
        
        // 处理构造器为type类型的Wrapper类
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}

private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();
		// 每个目录去加载实现类，真正加载实现类的地方
    Map<String, Class<?>> extensionClasses = new HashMap<>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    // 兼容dubbo贡献apache基金会之前的老版本配置
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```
至此，实现类的加载已经完成，代码看起来没什么波澜不惊，但是其中还包含着自动适配的内容，这里不再展开，后面会有单独章节讲解dubbo的自动适配




## 四.总结
- 1.本篇博客主要科普性质的介绍java spi和dubbo的spi机制，并介绍两者之间的区别和联系
- 2.可以建立对spi的服务发现机制有一个清晰的认识
- 3.感兴趣的同学可以再深挖一下dubbo的ExtensionLoader代码，里面很有意思，dubbo的SPI机制基本都在这个类中实现
