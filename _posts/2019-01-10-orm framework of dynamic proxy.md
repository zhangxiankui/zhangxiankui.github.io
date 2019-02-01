---
layout: post
title:  "持久层框架必备利器之动态代理"
categories: ORM框架 设计模式 mybatis
tags:  源码解析 代理模式
author: zhangxiankui
---

* content
{:toc}


## 一、前言
所谓层框架，其实就是指的目前开源的mybatis，hibernate，paoding-jade等，这一期主要聊聊这些持久层框架都会用到的一个技术-动态代理，并以mybatis为例，来解释
动态代理在mybatis中的应用

## 二、动态代理之代理模式
#### 1.代理模式
- 代理模式是java设计模式中的一种，代理模式的定义是这样的，所谓代理，就是指控制对某个对象的访问，以达到代理该对象的目的
- java里面是万物皆对象，那么生活中我们可以把12306售票比喻成一个对象，那么类似携程，去哪儿网,火车票的代售点就是12306的代理，代理里面可能做一些被
代理对象不会做的事，比如之前的国内早期的火车票代售点是会收取手续费的
- 代理模式又分为静态代理和动态代理，静态代理是指代理类在编译之前已经确定下来了，而动态代理是代理类是在程序运行过程中才生成的，
所以静态和动态是指的代理类生成的静态和动态，其中动态代理又分为JDK动态代理和cglib的动态代理，下面再来介绍一下两种代理技术

#### 2.静态代理模式
静态代理在平常代码中用的很少，主要还是因为静态代理所带来的代码量难以维护，但是代理可以封装业务逻辑处理，使业务方只需要跟代理合作就能达到想要的目的，
当然每个对象可以有很多代理，而且每个代理所拥有的权限还可以不一样，生活中很常见的，大家可以考虑怎么通过代码实现。。
```
/**
 * 静态代理类需要实现的接口类
 *
 * @ClassName: Person
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月24日 下午10:17:32
 *
 */
public interface Person
{
	/**
	 * 说话
	 * @Title: speak
	 * @Description: TODO(这里用一句话描述这个方法的作用)
	 * @return void    返回类型
	 * @throws
	 */
	void speak();
}

/**
 * 学生类（这里扮演被代理的角色）
 * @ClassName: Student
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月24日 下午10:19:26
 *
 */
public class Student implements Person
{
	@Override
	public void speak()
	{
		System.out.println("我是一个学生");
	}

}

/**
 * person的代理类
 * 说明：
 * 	  1.静态代理类需要实现同一个接口
 *    2.需要将被代理的对象传入代理类中，这样才能实现代理
 *    3.静态代理有两种实现方式，可以通过继承和实现来完成静态代理
 *
 * @ClassName: PersonProxy
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月24日 下午10:21:15
 *
 */
public class PersonProxy implements Person
{
	private Person person;
	public PersonProxy()
	{
	}

	public PersonProxy(Person person)
	{
		this.person = person;
	}

	@Override
	public void speak()
	{
		System.out.println("我正在代理" + person.getClass().getSimpleName() + "类");
		person.speak();
	}

}

/**
 * 静态代理的实现类
 * @ClassName: StaticProxyTest
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月24日 下午10:26:01
 *
 */
public class StaticProxyTest
{
	public static void main(String[] args)
	{
		// 1.创建需要被代理的对象
		Student student = new Student();
		// 2.创建代理对象，去完后才能对Student对象的代理
		PersonProxy personProxy = new PersonProxy(student);
		// 3.调用代理对象的方法
		personProxy.speak();
	}
}
```
上面代码就是某代理类代理一个对象在处理之前进行某项操作，上面也提到了静态代理的实现步骤和方法，下面是运行结果
```
我正在代理Student类
我是一个学生

Process finished with exit code 0
```

#### 3.动态代理
###### 1.JDK动态代理
我们先来看下如何为一个类创建JDK的动态代理
```
/**
 * 动态代理类需要实现的接口类
 * 
 * @ClassName: Person 
 * @Description: TODO(这里用一句话描述这个类的作用) 
 * @author xkzhang
 * @date 2017年9月24日 下午10:17:32 
 *
 */
public interface Person
{
	// 说话
	void speak();
}

/**
 * 学生类（这里扮演被代理的角色）
 * @ClassName: Student
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月24日 下午10:19:26
 *
 */
public class Student implements Person
{
	@Override
	public void speak()
	{
		System.out.println("我是一个学生");
	}

}

/**
 * JDK动态代理类
 *
 * 实现jdk动态代理都需要一个类来实现InvocationHandler，
 * 并重写它的invoke方法，然后在该方法中来进行代理需要进行的事情
 * 
 * @ClassName: PersonProxyDy
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月25日 下午10:12:12
 *
 */
public class JDKPersonProxyDy implements InvocationHandler
{
	private Student student;

	public JDKPersonProxyDy()
	{
		// TODO Auto-generated constructor stub
	}

	/**
	 * 必须要有一个入参是被代理接口的构造方法，将被代理对象传入
	 */
	public JDKPersonProxyDy(Student student)
	{
		this.student = student;
	}

	/**
	 * 这个方法是动态生成代理类真正执行的方法
	 *
	 * proxy：标识当前代理对象，所以这里的方法调用的实例
	 * method:处理器拦截到代理类的方法
	 * params：方法的参数
	 */
	@Override
	public Object invoke(Object proxy, Method method, Object[] params) throws Throwable
	{
		System.out.println("记录方法日志开始");
		return method.invoke(student, params);
	}

	/**
	 * 在main方法演示了如何完成动态代理
	 */
	public static void main(String[] args)
	{
		// 1.创建被代理的对象
		Student student = new Student();
		// 2.实例化实现了InvocationHandle接口的代理类，并将被代理对象注入到代理类中
		JDKPersonProxyDy dy = new JDKPersonProxyDy(student);

		// 3.利用Proxy类获取student的代理对象
		Person person = (Person) Proxy.newProxyInstance(dy.getClass().getClassLoader(), 
				Student.class.getInterfaces(), dy);

		// 4.利用代理对象调用speak方法
		person.speak();
	}

}

----下面的是打印的结果
记录方法日志开始
我是一个学生
```
说到这里，很多同学都会好奇，这个jdk动态代理到底是怎么完成运行时创建代理类的，其实很简单，你就想着为该接口创建一个实现类就行了，
然后再编译，用加载器实时加载刚刚生成的类，保证虚拟机能够访问到就行；
那么下面就为你解析，下面是之前手写的一份简单的jdk动态代理的代码,同学们可以拿着代码去试一下（这里可能会有一个问题，就是你的PC必须有一个安装版的jdk，不能是绿色版的），
或者优化更好的版本
```
/**
 * 手动实现JDK动态代理
 * @ClassName: JDKDynamicProxyCustomer
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月26日 下午9:48:55
 *
 */
public class JDKDynamicProxyCustomer
{
	/**
	 * @throws IOException
	 * 获取一个对象的代理类
	 *
	 *  1.在接口的同一目录下创建一个$proxy.java文件（文件内容为实现该接口的$proxy类）
	 *  2.编译$proxy.java文件，生成$Proxy.class文件
	 *  3.通过这个$Proxy.class文件创建一个该类的实例，这样便生成了实现该接口的代理类
	 *  4.编译加载该$Proxy.class，供虚拟机使用
	 * @Title: getProxy
	 * @Description: TODO(这里用一句话描述这个方法的作用)
	 * @param classLoader 加载器
	 * @param interfaces 被代理的接口
	 * @return    设定文件
	 * @return Object    返回类型
	 * @throws
	 */
	public static Object getProxy(ClassLoader classLoader,
								  Class<?> interfaces, InvocationHandler handler) throws IOException
	{
		// 换行符
		String newLine = "\r\n";

		/**
		 * 获取文件包路径
		 */
		String interfaceName = interfaces.getName();  // 全路径名称
		String packPath = interfaceName.substring(0, interfaceName.lastIndexOf("."));

		// 封装接口方法,调用InvocationHandle方法
		String methodStr = "";
		for (Method method : interfaces.getDeclaredMethods())
		{
			methodStr +=
					"	@Override" + newLine +
							"	public void " + method.getName() +"()" + newLine +
							"	{" + newLine +
							"		System.out.println(\"我是" + method.getName() + "方法\");"+ newLine +
							"		try"+ newLine +
							"		{"+ newLine +
							"		handle.invoke(this, " + interfaces.getSimpleName()
							+ ".class.getMethod(\"" + method.getName() + "\", null), null);"+ newLine +
							"		}"+ newLine +
							"		catch (Throwable e)"+ newLine +
							"		{"+ newLine +
							"		    e.printStackTrace();"+ newLine +
							"		}"+ newLine +
							"	}" + newLine;
		}

		/**
		 * 一、在接口的同一目录下创建一个$proxy.java文件
		 */
		// 1.生成文件内容
		String fileData =
				"package " + packPath + ";" + newLine +
						"import java.lang.reflect.InvocationHandler;" + newLine +
						"public class $Proxy implements " + interfaces.getSimpleName() + newLine +
						"{ " + newLine +
		/*"     private " + handler.getClass().getName() + " handle = new "
		+ handler.getClass().getName() +"(new com.zxk.proxy.staticproxy.Student()"
		+ ");" + newLine + // 创建InvocationHandler对象

*/
						"     private InvocationHandler handle;" + newLine + // 创建InvocationHandler对象
						"     public $Proxy(InvocationHandler handle)"  + newLine +
						"     {"  + newLine +
						"     	  this.handle = handle;"  + newLine +
						"     }"  + newLine +
						methodStr +
						"}";
		// 2.生成一个文件
		String fileName = System.getProperty("user.dir")
				+ "/bin/" + packPath.replace(".", "/") + "/$Proxy.java";
		File proxyFile = new File(fileName); // 文件名为当前工程名+工程classpath输出路径+接口的包路径
		// 3.将文件内容写入文件中
		BufferedWriter writer = new BufferedWriter(new FileWriter(proxyFile));
		writer.write(fileData);
		writer.flush();
		writer.close();

		/**
		 * 二、编译$Proxy.java文件
		 */
		// 1.获取当前Java编译器
		JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
		// 2.获取当前的文件管理器
		StandardJavaFileManager fileManager = compiler.getStandardFileManager(null, null, null);
		// 3.获取编译任务并执行编译任务
		Iterable compilationUnits = fileManager.getJavaFileObjects(fileName);
		CompilationTask task = compiler.getTask(null, fileManager, null, null, null, compilationUnits);
		task.call();

		/**
		 * 三、加载编译好的Class文件到jvm中
		 */
		Class<?> c = null;
		Object result = null;
		try
		{
			c = classLoader.loadClass(packPath + ".$Proxy");
			// 1.直接通过代理类得无参构造方法创建 这种方式不通用，当接口方法传入参数就不适用了
			/*result = c.newInstance();*/

			// 2.通过构造方法（InvocationHandler）创建对象实例，可以直接用这种带构造器的方法
			Constructor<?> constructor = c.getConstructor(InvocationHandler.class);
			result = constructor.newInstance(handler);
		}
		catch (Exception e)
		{
			System.out.println("加载.class异常");
			e.printStackTrace();
		}

		return result;
	}

	public static void main(String[] args) throws IOException
	{
		// 1.创建被代理的对象
		Student student = new Student();

		// 2.实例化实现了InvocationHandle接口的代理类，并将被代理对象注入到代理类中
		JDKPersonProxyDy dy = new JDKPersonProxyDy(student);

		// 3.通过自己的getProxy去获取代理对象$Proxy
		com.zxk.proxy.staticproxy.Person person = (com.zxk.proxy.staticproxy.Person)
				getProxy(com.zxk.proxy.staticproxy.Person.class.getClassLoader(), com.zxk.proxy.staticproxy.Person.class, dy);

		if (null != person)
		{
			person.speak();
		}
	}
}



// 代码生成的$Proxy类/*
package com.zxk.proxy.staticproxy;
import java.lang.reflect.InvocationHandler;
public class $Proxy implements Person
{ 
     private InvocationHandler handle;
     public $Proxy(InvocationHandler handle)
     {
     	  this.handle = handle;
     }
	@Override
	public void speak()
	{
		System.out.println("我是speak方法");
		try
		{
		handle.invoke(this, Person.class.getMethod("speak", null), null);
		}
		catch (Throwable e)
		{
		    e.printStackTrace();
		}
	}
}

运行main方法的结果为：
我是speak方法
记录方法日志开始
我是一个学生
```
这里没有对jdk动态代理做源码解析，不过大家可以根据Proxy进去追代码，会发现ProxyGenerator的generateClassFile方法是真正生成java文件的地方，就是我上面的那个方法

###### 2.CGLIB动态代理
下面看一下代码怎么创建一个cglib的动态代理的代理类
```
/**
 * 被代理类
 *
 * Created By IntelliJ IDEA
 * Author: zhangxiankui
 * DATA: 2017年9月25日 下午10:50:42
 */
public class Teacher
{
	void speak()
	{
		System.out.println("我是一名老师！");
	}
}

/**
 * Cglib动态代理类
 * @ClassName: CglibPersonProxyDy
 * @Description: TODO(这里用一句话描述这个类的作用)
 * @author xkzhang
 * @date 2017年9月25日 下午10:50:42
 *
 */
public class CglibPersonProxyDy implements MethodInterceptor
{
	private Enhancer enhancer = new Enhancer();

	public Object getProxy()
	{
		enhancer.setSuperclass(Teacher.class);
		enhancer.setCallback(this);
		return enhancer.create();
	}

	/**
	 * proxy:标识代理对象
	 * method：当前拦截的方法
	 * params：方法参数
	 * methodProxy：代理方法，用于生成的代理对象（子类）去调用父类的方法
	 */
	@Override
	public Object intercept(Object proxy, Method method, Object[] params,
							MethodProxy methodProxy) throws Throwable
	{
		System.out.println("记录日志开始");
		return methodProxy.invokeSuper(proxy, params);
	}

	public static void main(String[] args)
	{
		// 1.实例化代理类
		CglibPersonProxyDy cglibPersonProxyDy = new CglibPersonProxyDy();

		// 2.通过代理类的getProxy方法获取代理对象
		Teacher teacher = (Teacher) cglibPersonProxyDy.getProxy();

		// 3.通过获取的代理对象来调用speak方法
		teacher.speak();
	}

}
```
cglib是基于ASM字节码实现的，所以在JDK1.6之前，动态代理JDK是比CGLIB慢的，但是随着JDK的更新，
1.7,1.8的JDK动态代理会比CGLIB的效率更高，并不是人们普遍认为的CGLIB的速度比JDK代理快

## 三、mybatis面向接口式编程之动态代理
来看mybatis面向接口式编程代码
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
if (null != inputStream) {
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    UserService service = sqlSession.getMapper(UserService.class);
    List<User> list = service.getUserList();
    System.out.println(list.size());
} else {
    System.out.println("未找到核心配置文件！");
}

// service接口代码
public interface UserService {

    /**
     * 获取用户列表
     * @return
     */
    public List<User> getUserList();
}
```
从上面代码可以看出，mybatis其实是通过sqlSession的getMapper方法去获取接口UserService的一个代理，下面我们来看下mybatis到底是怎么做的
#### 1.SqlSession之getMapper
```
public <T> T getMapper(Class<T> type) {
  /**
   * 其实SqlSession的getMapper有两个实现类，但是现在进来的是DefaultSqlSession中，
   * 这要需要去看一下factory中创建时候的初始化类，感兴趣同学可以去看下
   */
  return configuration.<T>getMapper(type, this);
}
```
可以看出来，最终还是调的Configuration的getMapper方法，下面进到Configuration去看一下
```
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  /**
   * 这里又出现了一个新的类mapperRegistry，顾名思义，其实就是注册mapper的类，也就是说维护了mapper
   * 的代理类信息，接下来就可以到该类中取看下到底是怎么去获取代理类
   */
  return mapperRegistry.getMapper(type, sqlSession);
}
```
#### 2.MapperRegistry的getMapper方法
```
/**
 * 获取接口类型的代理类
 *  1.根据接口类型获取对应的代理工厂，需要注意的是这个代理工厂和接口类型的对应关系其实在
 *  mybatis初始化的时候就已经建立了，之前我们分析初始化源码的时候，在最后封装Configuration对象
 *  的时候就有一个mapperElement方法，这个方法就是封装mapper的，当时没有再往深入分析，里面代码
 *  其实很简单，大家可以自己去阅读一下
 *  2.根据代理工厂去生成代理对象
 *
 * @param type 接口类型
 * @param sqlSession  mybatis中非常重要的的类，用来执行sql语句，获取mapper等重要功能
 * @param <T> 接口泛型
 * @return
 */
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 1.获取代理对象
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    // 2.获取代理类，这里为什么要传入sqlSession呢？可以想到，代理类中肯定涉及到sql查询等操作，这个是必须的
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```
#### 3.MapperProxyFactory之newInstance
```
/**
 * 调用JDK的Proxy类获取代理类
 * @param mapperProxy
 * @return
 */
protected T newInstance(MapperProxy<T> mapperProxy) {
  /**
   * 这里是不是似曾相识？翻下上面JDK动态代理的代码就知道了
   * 而且我们可以猜到MapperProxy肯定会实现InvocationHandler接口，下面去解析一下MapperProxy
   */
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```
#### 4.解析MapperProxy
```
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        // 1.如果方法是父类Object的方法,就直接调用父类方法，如wait,notify,toString,hashCode等
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        // 2.如果是接口的默认方法（JDK1.8的新特性），那么调用默认方法
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }

    // 3.这里是真正执行接口的方法
    final MapperMethod mapperMethod = cachedMapperMethod(method); // 实例化MapperMethod，并进行缓存
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }

  /**
   * 由于接口不能实例化，所以需要依赖MethodHandles来进行方法调用
   * @param proxy
   * @param method
   * @param args
   * @return
   * @throws Throwable
   */
  private Object invokeDefaultMethod(Object proxy, Method method, Object[] args)
      throws Throwable {
    // 1.获取MethodHandles的内部类的参数为Class，int的构造器
    final Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class
        .getDeclaredConstructor(Class.class, int.class);

    // 2.设置构造器可访问
    if (!constructor.isAccessible()) {
      constructor.setAccessible(true);
    }

    // 通过MethodHandles绑定代理类proxy，并调用代理类的方法
    final Class<?> declaringClass = method.getDeclaringClass();
    return constructor
        .newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bindTo(proxy).invokeWithArguments(args);
  }

  /**
   * Backport of java.lang.reflect.Method#isDefault()
   *
   * 判断是否是接口的默认方法，下面解析一下如何判断，很有趣
   *   1.method.getModifiers()获取方法的语言修饰符的整数值（PUBLIC为1，STATIC为8，ABSTRACT为1024），当
   *   方法修饰符是public abstract时，该方法返回1025（1024+1），当和1033（1024+1+8）做位与操作时，当结果是
   *   public就能说明该方法不是抽象方法
   *   2.又因为方法的声明类是接口，那么显而易见，这个方法必定是JDK1.8新增的默认方法
   */
  private boolean isDefaultMethod(Method method) {
    return (method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC
        && method.getDeclaringClass().isInterface();
  }
}
```
#### 5.MapperMethod之execute
```
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  /**
   * 根据不同的sql类型来进行处理
   *    最终调用sqlSession来进行处理，这里我们后面再对sqlSession的处理做分析
   */
  switch (command.getType()) {
    case INSERT: {
  	Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional() &&
            (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

## 四、总结
- 动态代理对于ORM框架实现面向接口编程是必不可少的工具，所以建议大家要掌握这个工具，如果有感兴趣的同学了可以自己实现
一下这里的面向接口编程，这样会对动态代理有更深的认识，也会对整个ORM框架有深刻了解，有同学会说这样不就是重复造轮子嘛，
但是我认为如果你不去造轮子，你无法深入理解所用到的知识点