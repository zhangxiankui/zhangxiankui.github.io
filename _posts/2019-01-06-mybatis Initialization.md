---
layout: post
title:  "Mybatis初始化源码解析"
categories: Mybatis
tags:  源码解析
author: zhangxiankui
---

* content
{:toc}


## 一、前言
近期博主有走读mybatis源码的想法，虽然之前也看过，但是并没有对此做过笔记，所以近期会出一些关于mybatis的源码解析的博客，所以这边博客先从
Mybatis的初始化入手来进行源码解析

## 二、Mybatis初始化之SqlSessionFactoryBuilder
#### 1.下面是mybatis初始化的代码
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
// 通过SqlSessionFactoryBuilder对象去进行初始化
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
#### 2.下面进入到SqlSessionFactoryBuilder中看下build方法的处理
```
public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
}

/**
 *  该方法加载核心配置文件mybatis-config.xml，并封装返回SqlSessionFactory
 *
 * @param inputStream 这个是mybatis-config.xml的输入流
 * @param environment 这个是可以指定数据库连接的环境，也即是配置文件中的environment的id值
 * @param properties 可以指定某些全局的参数配置，相当于配置文件中的properties标签配置的值
 * @return
 */
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 1.这里是通过SAX去解析之前读到的数据流，并将得到的Document对象封装成XMLConfigBuilder对象
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);

    // 2.真正初始化的方法，通过读取到的配置文件的内容来封装Configuration对象
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```
上面可以看出，真正处理初始化的类是XMLConfigBuilder，下面进入XMLConfigBuilder中进行分析

## 三、Mybatis初始化之XMLConfigBuilder
#### 1.先来看一下该类的构造器
```
public XMLConfigBuilder(Reader reader, String environment, Properties props) {
  this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
}
```
可以看出，这里面又出现了两个新的类，分别是XMLMapperEntityResolver和XPathParser，下面分别解析这两个类
```
/**
 * Converts a public DTD(XSD) into a local one
 *  从这个英文很明显能看出来这里是将核心配置文件中的dtd/xsd转换成本地的dtd/xsd文件
 *
 *  本地的文件存放在org/apache/ibatis/builder/xml目录下,可以到该类去查看
 * @param publicId The public id that is what comes after "PUBLIC"
 * @param systemId The system id that is what comes after the public id.
 * @return The InputSource for the DTD(XSD)
 * 
 * @throws org.xml.sax.SAXException If anything goes wrong
 */
@Override
public InputSource resolveEntity(String publicId, String systemId) throws SAXException {
  try {
    if (systemId != null) {
      String lowerCaseSystemId = systemId.toLowerCase(Locale.ENGLISH);
      if (lowerCaseSystemId.contains(MYBATIS_CONFIG_SYSTEM) || lowerCaseSystemId.contains(IBATIS_CONFIG_SYSTEM)) {
        return getDtdInputSource(MYBATIS_CONFIG_DTD, publicId, systemId);
      } else if (lowerCaseSystemId.contains(MYBATIS_MAPPER_SYSTEM) || lowerCaseSystemId.contains(IBATIS_MAPPER_SYSTEM)) {
        return getDtdInputSource(MYBATIS_MAPPER_DTD, publicId, systemId);
      } else if (systemId.contains(MYBATIS_CONFIG)) {
        return getXsdInputSource(MYBATIS_CONFIG_XSD);
      } else if (systemId.contains(MYBATIS_MAPPER)){
        return getXsdInputSource(MYBATIS_MAPPER_XSD);
      }
    }
    return null;
  } catch (Exception e) {
    throw new SAXException(e.toString());
  }
}

/**
 * XPathParser的构造器
 */
public XPathParser(InputStream inputStream, boolean validation, Properties variables, EntityResolver entityResolver) {
  // 1.这里是做最常规的构造器封装，设置变量值
  commonConstructor(validation, variables, entityResolver);

  // 2.看见这个Document我们就很熟悉了，其实就是利用SAX解析获取XML文件的Document对象
  this.document = createDocument(new InputSource(inputStream));
}
```
这样就很清楚了，既然document对应已经拿到，下面必然会根据这个对象来进行Configuration对象的封装啦，

#### 2.解析Document对象，封装Configuration
```
/**
 * 包装Configuration对象
 * @return Configuration mybatis中最重要的配置对象闪亮登场
 */
public Configuration parse() {
  if (parsed) {
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;
  // 这里是实际真正包装的方法，这里的通过标签名从document中取标签信息的方法evalNode这里就不再展开，感兴趣可以自己去看
  parseConfiguration(parser.evalNode("*[local-name()='configuration']"));
  return configuration;
}
```

#### 3.XMLConfigBuilder之parseConfiguration
```
/**
 *  根据配置的信息来封装Configuration对象
 *
 * @param root 核心配置文件的根节点，表示configuration节点
 */
private void parseConfiguration(XNode root) {
  try {
    // 读取properties信息，通过resource或者url来加载属性信息；并将读取的信息赋值给Configuration对象的Variables属性
    propertiesElement(root.evalNode("*[local-name()='properties']"));

    // 读取settings，设置日志实现类和VFS
    Properties settings = settingsAsProperties(root.evalNode("*[local-name()='settings']"));
    loadCustomVfs(settings);
    loadCustomLogImpl(settings);

    // 读取typeAliases信息，并设置对象信息中
    typeAliasesElement(root.evalNode("*[local-name()='typeAliases']"));

    // 设置插件也可以称做拦截器
    pluginElement(root.evalNode("*[local-name()='plugins']"));

    // 配置对象创建的工厂，可用于创建实例
    objectFactoryElement(root.evalNode("*[local-name()='objectFactory']"));
    objectWrapperFactoryElement(root.evalNode("*[local-name()='objectWrapperFactory']"));
    reflectorFactoryElement(root.evalNode("*[local-name()='reflectorFactory']"));

    // 全局参数配置
    settingsElement(settings);

    // 设置环境信息，一般指数据库信息
    environmentsElement(root.evalNode("*[local-name()='environments']"));

    // 设置数据库产品名称，如mysql，oracle
    databaseIdProviderElement(root.evalNode("*[local-name()='databaseIdProvider']"));

    // 设置typeHanlder，typeHanlder就是解决数据库类型和java类型的转换问题，可以配置到resultMap对象的result标签中
    typeHandlerElement(root.evalNode("*[local-name()='typeHandlers']"));

    // 设置对象关系映射信息，指用户自定的*mapper.xml文件
    mapperElement(root.evalNode("*[local-name()='mappers']"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```
至此，mybatis的初始化已经完成了

## 四、总结
#### 1.正如上面，初始化过程也就是Configuration对象封装过程
#### 2.有的同学会说，项目中都是mybatis和spring集成的，加载方式是怎样的呢？其实加载方式都是一样，只是需要在外面包装一层，实现几个spring初始化的接口，本质是一样的
我们下面来看一下mybatis和spring集成的配置信息
```
<!-- 配置数据源 这里数据源我们就不多说了，使用的alibaba的数据连接池-->
<bean id = "dataSource" class="com.alibaba.druid.pool.DruidDataSource">
   <property name="url" value="${url}"></property>
   <property name="driverClassName" value="${driverClass}"></property>
   <property name="username" value="${username}"></property>
   <property name="password" value="${password}"></property>
   <property name="maxActive" value="10"></property>
   <property name="minIdle" value="2"></property>
</bean>

<!-- 配置mybatis的sqlSessionFactory对象 -->
<bean id = "sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<!-- 这里面是数据源和核心配置文件的地址，相当于设置属性 -->
  <property name="dataSource" ref="dataSource"></property>
  <property name="configLocation" value="classpath:mybatis/mybatis.xml"></property>
</bean>

<!-- 配置扫描映射文件的文件包，这个很好理解的，就扫描*mapper.xml文件 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="basePackage" value="com.taotao.mapper"></property>
</bean>
```
很多同学都知道需要配置SqlSessionFactoryBean，但是并不知道它有什么用，它是怎么实现文件初始化的，下面上代码
```
/* 
 * 这个是SqlSessionFactoryBean的类定义，是不是看见一个非常熟悉的接口，对，没错，
 * 就是InitializingBean，了解spring的bean生命周期的同学都是
 * 知道的，当类实现了这个接口之后，spring在初始化bean的时候就会调用afterPropertiesSet方法（在属性设置完毕之后） 
 */
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent>
```
下面我们来看一下这个类的afterPropertiesSet（有关bean生命周期可以自行百度或者翻spring源码）
```
public void afterPropertiesSet() throws Exception {
    Assert.notNull(this.dataSource, "Property 'dataSource' is required");
    Assert.notNull(this.sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    // 这里就是初始化的地方
    this.sqlSessionFactory = this.buildSqlSessionFactory();
}

/**
 * 从这个方法是不是看到了一个熟悉的对象xmlConfigBuilder，是的，万变不离其总，这个解析我觉得就可以到此为止了
 *
 * 该方法总有一些删减，主要是为了突出xmlConfigBuilder，感兴趣可以去看完整源码
 */
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    XMLConfigBuilder xmlConfigBuilder = null;
    Configuration configuration;
    if (this.configLocation != null) {
        xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), (String)null, this.configurationProperties);
        configuration = xmlConfigBuilder.getConfiguration();
    } else {
        if (logger.isDebugEnabled()) {
            logger.debug("Property 'configLocation' not specified, using default MyBatis Configuration");
        }

        configuration = new Configuration();
        configuration.setVariables(this.configurationProperties);
    }

    if (xmlConfigBuilder != null) {
        try {
            xmlConfigBuilder.parse();
            if (logger.isDebugEnabled()) {
                logger.debug("Parsed configuration file: '" + this.configLocation + "'");
            }
        } catch (Exception var23) {
            throw new NestedIOException("Failed to parse config resource: " + this.configLocation, var23);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    return this.sqlSessionFactoryBuilder.build(configuration);
}
```
