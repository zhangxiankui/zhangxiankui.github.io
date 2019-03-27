---
layout: post
title:  "�ֲ�ʽ��Ⱥ֮session��������"
categories: ��������Ⱥ session���� nginx redis
tags:  session Cookie spring-session
author: zhangxiankui
---

* content
{:toc}


## һ��ǰ��
����Ŀǰ������Ҫ�Ե�ǰ�ķ�������Ⱥ��Session���������ṩ������������Ա�ƪ������Ҫ���۷ֲ�ʽ��Ⱥ�����ٵ�Session�����Լ����漰������һЩSession��ص�����

## ��������Session
˵��Session����ôcookie�ǲ��ò�˵��

#### 1.Cookie
##### Cookie��ʲô��
Cookie�Ƿ���˲�����ͨ��response��header���ظ��ͻ��˲������ڿͻ��˵Ķ�����ͼ��Cookie������ֵ��Cookie��Ϊ�����汾�����·ֱ���version0��version1
![](https://zhangxiankui.github.io/imgs/session/cookieProp.jpg)

##### Cookie��Seesion�Ĺ�ϵ��
- ʵ���Ϸ�������Session������Ҫͨ��Cookie��ʵ�֣��������Cookie����ǰ�û���SessionIdͨ��request��header���߷���ˣ���ô��������޷�֪����ǰ�û���
�����ĸ�Session�ģ�
- ��Ȼ��������ǿ���ͨ�����水ť����������ֹCookie����ô�������ˣ����Cookie��ֹ����ô���أ���������û�취��ȡSessionId�ˣ����������аٶ�
![](https://zhangxiankui.github.io/imgs/session/setCookie.jpg)

##### Cookie��Seesion�Ĳ����ĸ���ԭ��
- ��֪�������û�������ôһ�����⣬Ϊʲô�����Cookie��Session����������������Ϊ���ھ�һ��������ʷ��Ȼ�ԣ���ʵ���ҿ�����
���Ǵ��ڵĸ���ԭ����Http����״̬�ԣ�Ҳ����˵������������ͨ��HttpЭ������¼�û���״̬�����Ծ���Ҫĳ�������������������£�
- ��Session��Cookie��������ȱ�㣬�������ȱ��Ͳ���һһ׸��������Ȥ���о����������͵��������в�ͬ�����ó�����ʹ��Cookie���С��������ѹ��������ͬʱ��ռ�ô���Session
�����ӷ�����ѹ�������Ƕ��ڸ߲������������Ч������ʵ��ӳ��һ���ܾ�������⣬û�������ļ�����ֻ��������ҵ�񳡾�������������ҵ�񳡾�������������չ����������

##### Cookie�����������
- Cookie�����������Ǵ�����ȫ�ԵĽǶȿ��ǵģ����羭���CXRF��������ʵ���ǽٳ��˿ͻ��˵�Cookie������ģ��ͻ��˷���
- ���������Cookie�������Ƶģ�����ͼ
![](https://zhangxiankui.github.io/imgs/session/cookie-limit.jpg)


#### 2.Session
##### ʲô��Session��
- �������ʣ�Session�Ǳ�ƪ���͵����ǣ�Session�Ĺٷ������ǻỰ���ƣ����ڴ洢�û���������Ϣ��ͨ������˵��Session��Ҳ���Ƿ����������Ĳ��洢��
�����������ڼ�¼�û���Ϣ�Ķ���
- Session�Ǵ洢�ڷ������ϵģ�׼ȷ��˵��Sevlet�����е�

##### Seesion�Ĳ����ĸ���ԭ��
����cookie�ͷ������Ľ����ɱ��ߣ�Session���˸�����Ľ����������ôSessionӦ�˶���

##### Session����������
- ��ͳ�������ĵ��������ʱ����Session�ǿ��Ժ��аԵ��ģ���Ϊ�����û���Session����һ̨�������ϣ��������κ����⣻
- ��������������ҵ���������վ������������ָ���������������̨����������������ʵ��������Ծ���Ҫ��������չ��Ҳ���Ǵ��������Ⱥ����ô�ͻ���������
�����˹�Session�ڷ�������Ⱥ֮ǰ�Ĺ�������


## ��.Session����Ŀǰ�Ľ������
#### 1.�������ؾ����豸ʵ��ճ�ԻỰ
##### F5����nginxʵ��ճ�ԻỰ
- ��νճ�ԻỰ��ָĳһ�������û�����ֻ�ᱻ�ַ����ض���ĳһ̨��˷������ϣ������Ϳ��Խ����Ⱥ�е�Session����
- ����������һ�����⣬������ĳһ̨������崻�����ô�ܶ��û�������Ҫ�������µ�¼�Ȳ������û�����ܲ���

#### 2.����Servlet������Session����
##### ����Web����������Tomcat�Ѿ�֧��Session����
- ʹ�������ǳ��򵥣�ֻ��Ҫ��tomcat�зſ���Ⱥ����<cluster>��������web.xml������<distributable></distributable>��ǩ����
![](https://zhangxiankui.github.io/imgs/session/tomcat-server-cluster.png)
![](https://zhangxiankui.github.io/imgs/session/tomcat-web-session-copy.png)
- ���ǻ���Web��������Session���ƴ������¼������⣻1.����Session������Ҫ������֮��ͨ�ţ����Դ�����ʱ 2.����Ҫ��һ�����⣬���ַ������ܱȽϲһ��������������Ⱥ
�м�ʮ̨����ʱ����ʱ��Session�������������Ľ��Ǿ޴��

#### 3.�������������Session����
##### ����Web����������չ��ʵ��
- ����tomcat�Ĳ��[memcached-session-manager](https://github.com/magro/memcached-session-manager)��[tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)
�Լ�Jetty���[Jetty-nosql-mencache](https://github.com/yyuu/jetty-nosql-memcached)��[jetty-session-redis](https://github.com/Ovea/jetty-session-redis)��������鶼������
�����github����ȥ����
- ���ַ�ʽ����һ�����⣬���ǵ���Ŀ��ʹ��û�в����web������������websphere��weblogic��resin�ȣ���ô�ͺ�������
##### �Զ���ʵ��Session����
�Զ���ʵ�ֲ������£������Դ�������ķ�ʽ��չ����
- 1.����һ��������filter����http��requestת��Ϊ�����Զ����request,��Ϥweb�������˳���ͬѧӦ�ö�֪������filterһ��Ҫ������web.xml�ļ��ĵ�һ��
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

        // ��ȡ�ͻ��˵�cookieЯ����sessionId
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
            // �ͻ��˵�һ�η��ʣ�����sessionid������sessionid���õ����ص�response��

            // ����sessionid �����+��ǰʱ���+xkzhang
            Random random = new Random();
            sid = random.nextInt(10000) + System.currentTimeMillis() + "-xkzhang";

            Cookie cookie = new Cookie("JSESSIONID", sid);
            cookie.setPath("/");
            cookie.setMaxAge(-1);
            response.addCookie(cookie);
        }

        // ʹ���Զ����request����������
        chain.doFilter(new RedisHttpServletRequestWrapper(request, response, sid, meta), resp);
    }

    public void init(FilterConfig config) throws ServletException {
        meta.setHost(config.getInitParameter(HOST));
        meta.setPort(Integer.parseInt(config.getInitParameter(PORT)));
        meta.setSeconds(Integer.parseInt(config.getInitParameter(SECONDS)));
    }

}
```
- 2.�̳�HttpServletRequestWrapper��ʵ��getSession��������ȡ�����Զ����session��������ͳ鼸����Ҫ�ķ���
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
         * ��дgetSession�������ĳɻ�ȡ�Զ����RedisHttpSession
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
- 3.ʵ��httpsession�ӿڣ���д������Ҫ�õ��ķ���������set get��Щ���ĳɴ�redis��ȡ
```
/**
 * Created By IntelliJ IDEA
 * Author: zhangxiankui
 * Date: 2019/2/24
 * Time: 16:52
 */
public class RedisHttpSession implements HttpSession {

    private Logger logger = LoggerFactory.getLogger(RedisHttpSession.class);

    private final long creationTime = System.currentTimeMillis(); // session�Ĵ���ʱ��

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
     * ��ȡsession���� ����ֻ֧��string���͵�value
     *   1.����Jedis�ͻ���
     *   2.����sessionId��ȡ��ǰ��session��Ϣ��������Ϣ������hash�����ݽṹ
     * @param s
     * @return
     */
    public Object getAttribute(String s) {
        logger.info("��ȡsession��{} �����ԣ�{}", sid, s);

        String value = JedisPoolUtil.hget("JSESSIONID:" + sid, s);

        logger.info("��ȡ������ֵΪ��{}", value);

        return value;
    }

    public void setAttribute(String s, Object o) {
        logger.info("����session��{} �����ԣ�{}Ϊ��{}", sid, s, o);

        if (o instanceof String) {
            JedisPoolUtil.hset("JSESSIONID:" + sid, s, (String) o);
        }

        logger.info("��������ֵ�ɹ�!");
    }
}
```

##### ʹ��spring��[spring-session](https://docs.spring.io/spring-session/docs/current/reference/html5/#httpsession-redis)
- 1.spring�����ļ�
```
<context:annotation-config />

<!-- ����properties�ļ� -->
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
    <property name="maxInactiveIntervalInSeconds" value="${redis.session.timeout}" />    <!-- session����ʱ��,��λ���� -->
</bean>

<!--LettuceConnectionFactory ��������redis-->
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

  <!-- ���filter Ҫ���ڵ�һ�� ע�������filter-name���ܸģ������������-->
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
- 3.spring-session�ķ���
����һ�������springʵ���ǲ��Ǻܼ�࣬�ǵģ�spring��ʵ�Ƕ��������Զ�ʵ�ֵ�һ�ַ�װ�����ҽ���spring���������ʵ����װ�أ��Ե����úܼ򵥣�û����Ҳ��spring
���������ǵģ�����Ĵ�������Ͳ�չ��˵������Ȥͬѧ���Խ�������ȥ��ϸ������ʵ���Ǻ�������Ĳ�����ͬ�������������ܾ���
- 4.�������н������ͼ��ʾ����������Ⱥ������ô���ʣ�Sessionid���ǲ����
![](https://zhangxiankui.github.io/imgs/session/result.jpg)

## ��.�ܽ�
- 1.��ƪ���;�Session������������ܽᣬ�˽���Session��Cookie���ֵ�ԭ���Լ�Session�����������Լ���ȱ��
- 2.��Session���������������У����漰��redis��nginx�ķ������Լ�web��������Ⱥ�������ʡ��
- 3.�����еĽ��������������ȱ��ģ�û��һ�������ķ�����ֻ�����ʺ�ҵ��ķ���
