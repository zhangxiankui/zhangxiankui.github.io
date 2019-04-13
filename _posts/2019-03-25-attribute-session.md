---
layout: post
title:  "分布式集群之session共享问题"
categories: 服务器集群 session共享 nginx redis
tags:  session Cookie spring-session
author: zhangxiankui
---

* content
{:toc}


## 一、前言
由于目前工作需要对当前的服务器集群的Session共享问题提供解决方案，所以本篇博客主要讨论分布式集群所面临的Session问题以及所涉及的其他一些Session相关的问题

## 二、主角Session
说到Session，那么cookie是不得不说的

#### 1.Cookie
##### Cookie是什么？
Cookie是服务端产生并通过response的header返回给客户端并保存在客户端的对象；下图是Cookie的属性值，Cookie分为两个版本，上下分别是version0和version1
![](https://zhangxiankui.github.io/imgs/session/cookieProp.jpg)

##### Cookie和Seesion的关系？
- 实际上服务器的Session管理需要通过Cookie来实现，如果不是Cookie将当前用户的SessionId通过request的header告诉服务端，那么服务端是无法知道当前用户是
属于哪个Session的；
- 当然，浏览器是可以通过下面按钮来设置来禁止Cookie，那么问题来了，如果Cookie禁止了怎么办呢？服务器就没办法获取SessionId了，这个大家自行百度
![](https://zhangxiankui.github.io/imgs/session/setCookie.jpg)

##### Cookie和Seesion的产生的根本原因？
- 不知道大家有没有想过这么一个问题，为什么会出现Cookie和Session这两个东西，我认为存在就一定会有历史必然性；其实在我看来，
它们存在的根本原因是Http的无状态性，也就是说服务器并不能通过Http协议来记录用户的状态，所以就需要某个第三方用来完成这件事；
- 而Session和Cookie都有其优缺点，具体的优缺点就不再一一赘述（感兴趣可研究），这样就导致它们有不同的适用场景；使用Cookie会减小服务器的压力，但是同时会占用带宽，Session
会增加服务器压力，但是对于高并发的请求很奏效，这其实反映出一个很经典的命题，没有完美的技术，只有适用于业务场景的完美技术，业务场景是驱动技术发展的真正动力

##### Cookie会带来的问题
- Cookie带来的问题是从它安全性的角度考虑的，比如经典的CXRF攻击，其实就是劫持了客户端的Cookie请求来模拟客户端访问
- 浏览器对于Cookie是有限制的，如下图
![](https://zhangxiankui.github.io/imgs/session/cookie-limit.jpg)


#### 2.Session
##### 什么是Session？
- 毫无疑问，Session是本篇博客的主角，Session的官方定义是会话控制，用于存储用户的属性信息；通常我们说的Session，也就是服务器产生的并存储在
服务器的用于记录用户信息的对象；
- Session是存储在服务器上的，准确的说是Sevlet容器中的

##### Seesion的产生的根本原因？
由于cookie和服务器的交互成本高，Session成了该问题的解决方案，那么Session应运而生

##### Session带来的问题
- 传统互联网的单机部署的时代，Session是可以横行霸道的，因为所有用户的Session都在一台服务器上，不存在任何问题；
- 随着整个互联网业务的崛起，网站访问量伴随着指数级别的增长，单台服务器不能满足访问的需求，所以就需要做横向扩展，也就是搭建服务器集群，那么就会引出今天
的主人公Session在服务器集群之前的共享问题


## 三.Session问题目前的解决方案
#### 1.依赖负载均衡设备实现粘性会话
##### F5或者nginx实现粘性会话
- 所谓粘性会话是指某一个特性用户请求只会被分发到特定的某一台后端服务器上，这样就可以解决集群中的Session问题
- 但是这会带来一个问题，当其中某一台服务器宕机，那么很多用户都必须要进行重新登录等操作，用户体验很不好

#### 2.依赖Servlet容器的Session复制
##### 部分Web服务器诸如Tomcat已经支持Session复制
- 使用起来非常简单，只需要在tomcat中放开集群配置<cluster>，并且在web.xml中增加<distributable></distributable>标签即可
 ![](https://zhangxiankui.github.io/imgs/session/tomcat-server-cluster.png)
 
 ![](https://zhangxiankui.github.io/imgs/session/tomcat-web-session-copy.png)
- 但是基于Web服务器的Session复制存在以下几个问题；1.由于Session复制需要服务器之间通信，所以存在延时 2.最主要的一个问题，这种方案性能比较差，一旦整个服务器集群
有几十台机器时，此时的Session带来的性能消耗将是巨大的

#### 3.依赖三方缓存的Session共享
##### 基于Web服务器的扩展来实现
- 比如tomcat的插件[memcached-session-manager](https://github.com/magro/memcached-session-manager)和[tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)
以及Jetty插件[Jetty-nosql-mencache](https://github.com/yyuu/jetty-nosql-memcached)和[jetty-session-redis](https://github.com/Ovea/jetty-session-redis)，插件详情都可以在
上面的github上面去查找
- 这种方式存在一种问题，就是当项目中使用没有插件的web服务器，诸如websphere，weblogic，resin等，那么就很尴尬了
##### 自定义实现Session共享
自定义实现步骤如下，下面以代码解析的方式来展开：
- 1.创建一个拦截器filter，将http的request转化为我们自定义的request,熟悉web服务加载顺序的同学应该都知道，该filter一定要配置在web.xml文件的第一个

```
public class SessionFilter implements Filter {

    protected final Logger logger = LoggerFactory.getLogger(SessionFilter.class);

    private static SessionMeta meta = new SessionMeta();

    private static final String HOST ="host";

    private static final String PORT ="port";

    private static final String SECONDS="seconds";

    public void destroy() {
    }

    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;

        // 获取客户端的cookie携带的sessionId
        String sid = null;
        Cookie[] cookies = request.getCookies();
        if (null != cookies) {
            for (Cookie cookie : cookies) {
                if (cookie.getName().equals("JSESSIONID")) {
                    sid = cookie.getValue();
                }
            }
        }

        if (null == sid) {
            // 客户端第一次访问，创建sessionid，并将sessionid设置到返回的response中

            // 生成sessionid 随机数+当前时间戳+xkzhang
            Random random = new Random();
            sid = random.nextInt(10000) + System.currentTimeMillis() + "-xkzhang";

            Cookie cookie = new Cookie("JSESSIONID", sid);
            cookie.setPath("/");
            cookie.setMaxAge(-1);
            response.addCookie(cookie);
        }

        // 使用自定义的request来处理请求
        chain.doFilter(new RedisHttpServletRequestWrapper(request, response, sid, meta), resp);
    }

    public void init(FilterConfig config) throws ServletException {
        meta.setHost(config.getInitParameter(HOST));
        meta.setPort(Integer.parseInt(config.getInitParameter(PORT)));
        meta.setSeconds(Integer.parseInt(config.getInitParameter(SECONDS)));
    }

}
```


- 2.继承HttpServletRequestWrapper，实现getSession方法，获取我们自定义的session对象，这里就抽几个重要的方法

```
/**
 * Created By IntelliJ IDEA
 * Author: zhangxiankui
 * Date: 2019/2/27
 * Time: 21:57
 */
public class RedisHttpServletRequestWrapper extends HttpServletRequestWrapper {

    protected final Logger logger = LoggerFactory.getLogger(RedisHttpServletRequestWrapper.class);

    private RedisHttpSession currentSession;

    private HttpServletRequest request;

    private HttpServletResponse response;

    private String sid = "";

    private SessionMeta meta;

    public RedisHttpServletRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    public RedisHttpServletRequestWrapper(HttpServletRequest request, HttpServletResponse response,
                                          String sid, SessionMeta sessionMeta) {

        super(request);
        this.request = request;
        this.response = response;
        this.sid = sid;
        this.meta = sessionMeta;
    }

    @Override
    public HttpSession getSession(boolean create) {
        /**
         * 重写getSession方法，改成获取自定义的RedisHttpSession
         */
        if (null != currentSession) {
            return currentSession;
        }

        if (!create) {
            return null;
        }

        return new RedisHttpSession(sid, meta, request, response);
    }

    @Override
    public HttpSession getSession() {
        return getSession(true);
    }
}

```

- 3.实现httpsession接口，重写我们需要用到的方法，比如set get这些，改成从redis获取

```
/**
 * Created By IntelliJ IDEA
 * Author: zhangxiankui
 * Date: 2019/2/24
 * Time: 16:52
 */
public class RedisHttpSession implements HttpSession {

    private Logger logger = LoggerFactory.getLogger(RedisHttpSession.class);

    private final long creationTime = System.currentTimeMillis(); // session的创建时间

    private String sid = "";

    private HttpServletRequest request;

    private HttpServletResponse response;

    private final long lastAccessedTime = System.currentTimeMillis();

    private SessionMeta meta;

    public RedisHttpSession() {

    }

    public RedisHttpSession(String sid,SessionMeta meta, HttpServletRequest request,

                              HttpServletResponse response) {

        this.sid=sid;

        this.request=request;

        this.response=response;

        this.meta=meta;

    }

    public long getCreationTime() {
        return creationTime;
    }

    public String getId() {
        logger.info(getClass()+"getId():"+sid);
        return sid;
    }
    /**
     * 获取session属性 暂且只支持string类型的value
     *   1.创建Jedis客户端
     *   2.根据sessionId获取当前的session信息的属性信息，采用hash的数据结构
     * @param s
     * @return
     */
    public Object getAttribute(String s) {
        logger.info("获取session：{} 的属性：{}", sid, s);

        String value = JedisPoolUtil.hget("JSESSIONID:" + sid, s);

        logger.info("获取到属性值为：{}", value);

        return value;
    }

    public void setAttribute(String s, Object o) {
        logger.info("设置session：{} 的属性：{}为：{}", sid, s, o);

        if (o instanceof String) {
            JedisPoolUtil.hset("JSESSIONID:" + sid, s, (String) o);
        }

        logger.info("设置属性值成功!");
    }
}
```

##### 使用spring的[spring-session](https://docs.spring.io/spring-session/docs/current/reference/html5/#httpsession-redis)
- 1.spring配置文件

```
<context:annotation-config />

<!-- 加载properties文件 -->
<bean id="configProperties"
      class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:session-redis.properties</value>
        </list>
    </property>
</bean>

<!-- RedisHttpSessionConfiguration -->
<bean
        class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
    <property name="maxInactiveIntervalInSeconds" value="${redis.session.timeout}" />    <!-- session过期时间,单位是秒 -->
</bean>

<!--LettuceConnectionFactory 用于连接redis-->
<bean
        class="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory"
        p:host-name="${redis.host}" p:port="${redis.port}" p:password="${redis.pass}" />
```


- 2.web.xml

```
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath*:spring-session.xml</param-value>
  </context-param>

  <!-- 这个filter 要放在第一个 注意这里的filter-name不能改，否则会有问题-->
  <filter>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
  </filter>
  <filter-mapping>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>ERROR</dispatcher>
  </filter-mapping>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
```
- 3.spring-session的分析<br>
看了一下这里的spring实现是不是很简洁，是的，spring其实是对我上面自定实现的一种封装，并且借助spring容器来完成实例的装载，显得配置很简单，没错，这也是spring
所赋予我们的，具体的代码这里就不展开说，感兴趣同学可以进到里面去仔细看，其实就是和我上面的步骤相同，但是里面代码很精彩
- 4.具体运行结果如下图所示，服务器集群不管怎么访问，Sessionid都是不变的
![](https://zhangxiankui.github.io/imgs/session/result.jpg)

## 四.总结
- 1.本篇博客就Session相关问题做了总结，了解了Session和Cookie出现的原因以及Session问题解决方案以及优缺点
- 2.在Session共享问题解决方案中，会涉及到redis和nginx的服务器以及web服务器集群搭建，这里省略
- 3.这里列的解决方案都是有优缺点的，没有一个完美的方案，只有最适合业务的方案
