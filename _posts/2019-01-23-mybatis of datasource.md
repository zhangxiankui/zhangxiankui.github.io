---
layout: post
title:  "Mybatis之数据源处理"
categories: mybatis 事务 设计模式
tags:  源码解析 数据源 模板模式 策略模式 单一职责
author: zhangxiankui
---

* content
{:toc}


## 一、前言
本篇博客主要数据源角度来解读mybatis的源码，并了解mybatis的数据源如何配置，mybatis对数据源进行了哪些优化，以及mybatis对数据源加载的支持类型，
当然，说到数据源，事务是不得不提的，会基于mybatis的事务处理来简单讨论事务（放至spring事务中一块分析）

## 二、初始化数据环境
- 数据源必须要从初始化说起，看下mybatis到底是如何进行初始化，以及支持哪几种加载数据源的方式，基于前面mybatis初始化那篇博客，我们从XMLConfigBuilder
这个类开始说起
- 这次再来看XMLConfigBuilder对象时，我有了新的体验，想到mybatis肯定还是有基于注解的初始化，那么必定有基于注解的配置编译器；那么mybatis是怎么做的呢？
```
/**
 * 可以看见继承自BaseBuilder，查看BaseBuilder之后会发现，里面都是一些
 *  比较重要的对象和方法，比如定义了Configuration对象等，没错，这里就是
 *  使用了模板模式，这样就使得子类可以保证可变性，而且公共的方法可以放到子类，
 *  提升代码整体的整洁度，并且提升代码的可扩展性，是一个面向扩展的设计模式
 *
 * @author Clinton Begin
 * @author Kazuki Shimizu
 */
public class XMLConfigBuilder extends BaseBuilder {
```
#### 1.XMLConfigBuilder之environmentsElement
```
/**
 * 初始化Environment
 * @param context
 * @throws Exception
 */
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
        // 获取默认的环境
      environment = context.getStringAttribute("default");
    }
    for (XNode child : context.getChildren()) {
      String id = child.getStringAttribute("id");
      // isSpecifiedEnvironment这个方法很有意思，判断id是否和默认id相同，也就是说mybatis只初始化默认environment
      if (isSpecifiedEnvironment(id)) {
        // 初始化事务管理器
        TransactionFactory txFactory = transactionManagerElement(child.evalNode("*[local-name()='transactionManager']"));
        // 初始化数据源，通过工厂对DataSource进行包装
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("*[local-name()='dataSource']"));
        DataSource dataSource = dsFactory.getDataSource();
        // Environment中包含唯一id、事务管理器、数据源
        Environment.Builder environmentBuilder = new Environment.Builder(id)
            .transactionFactory(txFactory)
            .dataSource(dataSource);
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
```
可以看出，mybatis的environment包含了事务处理器和数据源，下面来一一解析事务处理器和数据源的加载

#### 2.XMLConfigBuilder初始化事务处理器之transactionManagerElement
```
private TransactionFactory transactionManagerElement(XNode context) throws Exception {
  if (context != null) {
    // 获取事务管理的类型
    String type = context.getStringAttribute("type");
    // 获取事务管理器的属性
    Properties props = context.getChildrenAsProperties();
    // 根据类型创建TransactionFactory，具体别名类型在Configuration对象的初始化中定义
    TransactionFactory factory = (TransactionFactory) resolveClass(type).newInstance();
    factory.setProperties(props);
    return factory;
  }
  throw new BuilderException("Environment declaration requires a TransactionFactory.");
}
```
这里面有涉及到解析xml的相关的内容，感兴趣的可以进去深究；这里面比较难理解的是resolveClass这个方法，其实
就是通过对应别名找class对象，进入该方法看一下
```
/**
 * 父类封装的公共方法，供子类调用
 * @param alias
 * @param <T>
 * @return
 */
protected <T> Class<? extends T> resolveClass(String alias) {
  if (alias == null) {
    return null;
  }
  try {
    // 解决别名映射
    return resolveAlias(alias);
  } catch (Exception e) {
    throw new BuilderException("Error resolving class. Cause: " + e, e);
  }
}
```
自己翻源码肯定会发现，这个方法其实是在父类BaseBuilder中的，现在大家应该体会到mybatis的coder为什么设计代码
为这种子父类关系了把，这就是好代码，mark一下
```
protected <T> Class<? extends T> resolveAlias(String alias) {
  // typeAliasRegistry，顾名思义，这个类就是来进行类型别名注册的
  return typeAliasRegistry.resolveAlias(alias);
}
```
- 上面出现了一个typeAliasRegistry类，自己动手翻源码就知道这个就是维护类型和别名的关系的，并且提供两者关系注册和查询的方法，这又让我
不禁想起了设计模式里面的一个原则，没错，就是单一职责原则；虽然类很多，但是整个业务逻辑就会变得很清晰，在代码修改过程中并不会对其他
模块造成问题
- 另外一个问题，这里面突然冒出来一个typeAliasRegistry，它是从哪冒出来的呢，注意到之前初始化系列文章的肯定知道，是在XmlConfigBuilder对象初始化时候设置进来的，
而且typeAliasRegistry其实是Configuration的对象，并转交给BaseBuilder来进行处理
```
public Configuration() {
  /**
   * 注册事务管理器别名
   *    JDBC表示JdbcTransactionFactory，意味着事务管理交由JDBC
   *    MANAGED表示ManagedTransactionFactory，意味着事务管理交由mybatis处理
   */
  typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
  typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);
```
这么一看，是不是就清楚了为什么事务处理器的Type要配置JDBC和MANAGED了
#### 3.TypeAliasRegistry之resolveAlias
```
/**
 * 查找别名对应的class对象
 * @param string 别名
 * @param <T> 别名对应的class对象
 * @return
 */
public <T> Class<T> resolveAlias(String string) {
  try {
    if (string == null) {
      return null;
    }
    // 转换成小写
    String key = string.toLowerCase(Locale.ENGLISH);

    /**
     * 从别名列表中查询别名对应的class
     *   如果缓存列表中不存在，就会以当前字符串做为
     *   全路径名来查找对应的class对象，也就是下面的classForName方法
     *
     *   TYPE_ALIASES对象没什么神奇的，就是一个Map对象，存放关系
     */
    Class<T> value;
    if (TYPE_ALIASES.containsKey(key)) {
      value = (Class<T>) TYPE_ALIASES.get(key);
    } else {
      value = (Class<T>) Resources.classForName(string);
    }
    return value;
  } catch (ClassNotFoundException e) {
    throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
  }
}
```
既然看到这里，我还得给你看一下数据库type和java类型的映射关系的定义,当然这是默认的，你也可以自定义，利用map特性重写就行
```
public TypeAliasRegistry() {
  registerAlias("string", String.class);

  registerAlias("byte", Byte.class);
  registerAlias("long", Long.class);
  registerAlias("short", Short.class);
  registerAlias("int", Integer.class);
  registerAlias("integer", Integer.class);
  registerAlias("double", Double.class);
  registerAlias("float", Float.class);
  registerAlias("boolean", Boolean.class);

  registerAlias("byte[]", Byte[].class);
  registerAlias("long[]", Long[].class);
  registerAlias("short[]", Short[].class);
  registerAlias("int[]", Integer[].class);
  registerAlias("integer[]", Integer[].class);
  registerAlias("double[]", Double[].class);
  registerAlias("float[]", Float[].class);
  registerAlias("boolean[]", Boolean[].class);

  registerAlias("_byte", byte.class);
  registerAlias("_long", long.class);
  registerAlias("_short", short.class);
  registerAlias("_int", int.class);
  registerAlias("_integer", int.class);
  registerAlias("_double", double.class);
  registerAlias("_float", float.class);
  registerAlias("_boolean", boolean.class);

  registerAlias("_byte[]", byte[].class);
  registerAlias("_long[]", long[].class);
  registerAlias("_short[]", short[].class);
  registerAlias("_int[]", int[].class);
  registerAlias("_integer[]", int[].class);
  registerAlias("_double[]", double[].class);
  registerAlias("_float[]", float[].class);
  registerAlias("_boolean[]", boolean[].class);

  registerAlias("date", Date.class);
  registerAlias("decimal", BigDecimal.class);
  registerAlias("bigdecimal", BigDecimal.class);
  registerAlias("biginteger", BigInteger.class);
  registerAlias("object", Object.class);

  registerAlias("date[]", Date[].class);
  registerAlias("decimal[]", BigDecimal[].class);
  registerAlias("bigdecimal[]", BigDecimal[].class);
  registerAlias("biginteger[]", BigInteger[].class);
  registerAlias("object[]", Object[].class);

  registerAlias("map", Map.class);
  registerAlias("hashmap", HashMap.class);
  registerAlias("list", List.class);
  registerAlias("arraylist", ArrayList.class);
  registerAlias("collection", Collection.class);
  registerAlias("iterator", Iterator.class);

  registerAlias("ResultSet", ResultSet.class);
}
```
至此，事务管理器的初始化就完成了，从上面可以看出，当你配置了JDBC时，就会创建JdbcTransactionFactory对象作为事务管理器，如果配置
MANAGE那么就会创建ManagedTransactionFactory对象，至于事务管理方面其实还是可以深究的，这里我们不展开，留到后面spring源码分析中解析

#### 4.数据源初始化解析之dataSourceElement
```
/**
 *  这个方法和事务管理器初始化很像，只是初始化的是数据源工厂
 * @param context
 * @return
 * @throws Exception
 */
private DataSourceFactory dataSourceElement(XNode context) throws Exception {
  if (context != null) {
    String type = context.getStringAttribute("type");
    Properties props = context.getChildrenAsProperties();
    DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
    factory.setProperties(props);
    return factory;
  }
  throw new BuilderException("Environment declaration requires a DataSourceFactory.");
}
```
可以看出这里和前面的事务管理器加载很相似，主要是看用户配置的数据源的别名是什么，那么别名分别对应什么呢？其实很简单，根据上面的经验，别名注册也肯定在
Configuration对象的初始化时完成
```
/**
 * 注册数据源工厂别名
 *   JNDI表示配置在tomcat的数据源,通过lookup去寻找，这种一般用的较少，除非在一些CMS系统中
 *   POOLED表示池化的数据源,也就是说实现了数据连接池的数据工厂
 *   UNPOOLED表示非池化的数据源
 */
typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);
```
对于这里我还是想多深入分析一下数据源，这里面重要的还是数据工厂怎么去产生数据源，每个数据工厂都会对应一个特定的数据源，下面通过DataSource来深入解析（数据工厂没什么可说的）
```
@Override
public Connection getConnection() throws SQLException {
  return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
}
```
熟悉DataSource的同学都知道，最重要的就是getConnection方法，它是最终对外提供数据库连接的接口；可以从这里来作为突破口看一下mybatis到底是怎么玩的
```
/**
 * 从连接池获取数据连接对象
 *
 * @param username 数据库用户名
 * @param password 密码
 * @return
 * @throws SQLException
 */
private PooledConnection popConnection(String username, String password) throws SQLException {
  boolean countedWait = false;
  PooledConnection conn = null;
  long t = System.currentTimeMillis();
  int localBadConnectionCount = 0;

  while (conn == null) {
    // 先锁住整个连接池对象，这个时候任何从数据连接池获取和释放数据库连接的线程都会挂起
    synchronized (state) {
      // 1.如果空闲列表有当前空闲的数据库连接，直接取第0位数据，归还的连接优先入空闲列表
      if (!state.idleConnections.isEmpty()) {
        // Pool has available connection
        conn = state.idleConnections.remove(0);
        if (log.isDebugEnabled()) {
          log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
        }
      } else {
        // 2.如果连接池未满，创建一个新的数据库连接
        if (state.activeConnections.size() < poolMaximumActiveConnections) {
          // Can create new connection
          conn = new PooledConnection(dataSource.getConnection(), this);
          if (log.isDebugEnabled()) {
            log.debug("Created connection " + conn.getRealHashCode() + ".");
          }
        } else {
          // 拿到最老的一个数据库连接
          PooledConnection oldestActiveConnection = state.activeConnections.get(0);
          long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
          // 判断当前数据库的连接时间是否超时，超时就将该数据库连接从池中去除，并创建新连接
          if (longestCheckoutTime > poolMaximumCheckoutTime) {
            // Can claim overdue connection
            state.claimedOverdueConnectionCount++;
            state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
            state.accumulatedCheckoutTime += longestCheckoutTime;
            state.activeConnections.remove(oldestActiveConnection);
            if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
              try {
                oldestActiveConnection.getRealConnection().rollback();
              } catch (SQLException e) {
                /*
                   Just log a message for debug and continue to execute the following
                   statement like nothing happened.
                   Wrap the bad connection with a new PooledConnection, this will help
                   to not interrupt current executing thread and give current thread a
                   chance to join the next competition for another valid/good database
                   connection. At the end of this loop, bad {@link @conn} will be set as null.
                 */
                log.debug("Bad connection. Could not roll back");
              }  
            }
            conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
            conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
            conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
            oldestActiveConnection.invalidate();
            if (log.isDebugEnabled()) {
              log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
            }
          } else {
            // 当最先获取的连接的超时时间没有大于连接设置的最大超时时间，那么必须等待
            try {
              if (!countedWait) {
                state.hadToWaitCount++; // 当前等待获取连接的线程数
                countedWait = true; // 是否有正在等待获取连接的线程
              }
              if (log.isDebugEnabled()) {
                log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
              }
              long wt = System.currentTimeMillis();

              /**
               * 当没获取到连接时等待20秒,不了解wait作用的可以自己去看下多线程相关内容
               *
               *  这里我还是要多说一句，线程被唤醒之后会继续执行后面的while循环，直到获取到连接为止
               *  所以在程序中可能会因为没有拿到数据库连接陷入无限期的阻塞状态，那么mybatis考虑到
               *  这个问题，在Datasource中可以设置对应连接池连接条数，这样具体可以根据业务来进行对应的配置
               */
              state.wait(poolTimeToWait);
              state.accumulatedWaitTime += System.currentTimeMillis() - wt;  // 统计连接池总共等待时间
            } catch (InterruptedException e) {
              break;
            }
          }
        }
      }
      if (conn != null) {
        // ping to server and check the connection is valid or not
        if (conn.isValid()) {
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
          conn.setCheckoutTimestamp(System.currentTimeMillis());
          conn.setLastUsedTimestamp(System.currentTimeMillis());
          state.activeConnections.add(conn);
          state.requestCount++;
          state.accumulatedRequestTime += System.currentTimeMillis() - t;
        } else {
          if (log.isDebugEnabled()) {
            log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
          }
          state.badConnectionCount++;
          localBadConnectionCount++;
          conn = null;
          if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
            if (log.isDebugEnabled()) {
              log.debug("PooledDataSource: Could not get a good connection to the database.");
            }
            throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
          }
        }
      }
    }

  }

  if (conn == null) {
    if (log.isDebugEnabled()) {
      log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }
    throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
  }

  return conn;
}
```
这是获取数据库连接的方法，可以看出实现和普通的连接池差不多，里面有涉及到多线程处理，但是我认为mybatis有一个很高明的地方，就是在数据连接的处理上，
它使用了动态代理，没错，就是动态代理，有人会说，这里用动态代理干嘛，我理解的它真正的用处是用代理类代理真正的数据库连接，然后提供给线程来进行数据库访问，
这样做的好处是，每次关闭的都只是代理对象，真正的数据库连接对象一直都在，也就是说可以减小应用数据库建连的时间，下面我们来看PooledCoonection
```
class PooledConnection implements InvocationHandler {
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();

    // 当调用代理类的close方法时，便会触发归还该数据库连接的操作，有兴趣同学可以研究一下pushConnection方法时怎么处理的
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
      dataSource.pushConnection(this);
      return null;
    }
    try {
      if (!Object.class.equals(method.getDeclaringClass())) {
        // issue #579 toString() should never fail
        // throw an SQLException instead of a Runtime
        checkConnection();
      }
      return method.invoke(realConnection, args);
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    
  }
}
```
这个动态代理的最重要就是当关闭数据库连接时，并不会真正关闭应用和数据库建立的连接，而是回收代理对象，的确很高明啊


#### 5.策略模式应用
本来这篇博客说到这也就结束了，但是我还是要把这里好代码给mark一下，也就是标题所说的策略模式，下面看一下使用策略模式的代码，其实也就上面的代码
```
DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
```
所谓策略模式，就是根据不同实现方法都能完成同一件事，这里的不同实现方法就是type，就是我们这里的变量，每一种type表示一种方式，比如pooled，jdni，unpooled等，
而实现的同一件事指的是生成DataSourceFactory，并通过它来完成获取数据源的操作，每一个变量都是一个实现类；这样实现的好处是代码的扩展性很好，代码易维护同学们可以考虑在
日常业务代码中使用

## 三.总结
- 1.通过这篇博客，我们可以深刻理解mybatis支持的数据源、事务处理器类型，以及对应数据源的作用和实现方式，为后面使用mybatis选择数据源提供很好的基础
- 2.当然我们也应该想到，其实我们自己是可以自定义自己的数据源和事务处理器类型的，然后再配置文件中配置自定义的处理器，也就是说实现DataSourceFactory和TransactionFactory
- 3.在读源码的过程中，我们应该从源码中取挖掘好代码，以便在后面的业务代码中可以使用到这些，那样才能提升自己的代码水平
