---
layout: post
title:  "类加载器之getResourceAsStream源码解析"
categories: ClassLoader 文件加载
tags:  文件加载 JDK源码 源码解析
author: zhangxiankui
---

* content
{:toc}


## 一、前言
为了搞清楚项目文件加载机制，所以这里对类加载器加载文件的方法做源码解析

## 二、类加载器简介
#### 1.由于后面源码分析中会用到类加载器相关知识，这里先简单介绍加载器
#### 2.加载器是jvm虚拟机用来加载class文件的对象，也就是通过ClassLoader类中的loadClass方法进行加载（对于加载器如何进行class文件加载这里不展开）,那么可以想到它必然就会存在寻找磁盘文件的方法
#### 3.JVM加载器的继承关系如下图所示：
![](https://zhangxiankui.github.io/imgs/originSource/jdk/ClassLoader-extends-picture.jpg)
图解：图中箭头方向就是继承的方向，ClassLoader是一个抽象类，是所有加载器的父类，当然除了根加载器bootstrap，另外用户自定义的加载器也可以直接
继承ClassLoader来实现自定义的class文件加载，例如JSPClassLoader就是继承ClassLoade，并通过为每个jsp对应类生成一个加载器来完成jsp页面的热加载的
（这里不展开，可以自行百度或者自己实现）另：这里面在ClassLoader和URLClassLoader类之间其实还有一个中间类SecureClassLoader，由于这里对源码分析无影响，
不再赘述


## 三.类加载器之getResourceAsStream
#### 1.getResourceAsStream方法
其实该方法被Classloader的实现类URLClassLoader重写了，两个方法都是调用ClassLoader的getResource方法
```
// 父类ClassLoader  
public InputStream getResourceAsStream(String name) {
        URL url = getResource(name);
        try {
            return url != null ? url.openStream() : null;
        } catch (IOException e) {
            return null;
        }
    }
    
// 子类URLClassLoader重写的方法
public InputStream getResourceAsStream(String name) {
        URL url = getResource(name);
        try {
            if (url == null) {
                return null;
            }
            // 这里两行代码就相当于父类中的url.openStream()方法，大家可以进源码看
            URLConnection urlc = url.openConnection();
            InputStream is = urlc.getInputStream();
            
            // 这里是不同的实现部分 *
            if (urlc instanceof JarURLConnection) {
                JarURLConnection juc = (JarURLConnection)urlc;
                JarFile jar = juc.getJarFile();
                synchronized (closeables) {
                    if (!closeables.containsKey(jar)) {
                        closeables.put(jar, null);
                    }
                }
            } else if (urlc instanceof sun.net.www.protocol.file.FileURLConnection) {
                synchronized (closeables) {
                    closeables.put(is, null);
                }
            }
            return is;
        } catch (IOException e) {
            return null;
        }
    }
```
- 对于这里存在一个疑问，为什么这里需要在父类ClassLoader中和子类URLClassLoader中都定义getResourceAsStream方法并且实现还存在区别，
这个问题是这样，由于ClassLoader相当于是JDK对外开放供外部实现加载器的接口，所以当前流行的框架都会实现该接口，那么当通过自定义加载器（未重写getResourceAsStream）
来进行文件查找时，会调用父类ClassLoader的方法，这就是为什么ClassLoader需要实现getResourceAsStream的理由
- 根据类加载的双亲委派模型，JVM中大多数类都是有扩展类加载器（EXT）和应用类加载器（APP），但是从上面的类继承关系分析可以看出，
这两个的父类都是URLClassLoader，所以在进行文件检索时，都会调用URLClassLoader的getResourceAsStream，这里肯定会有同学会觉得，为什么要在URLClassLoader
中重写，直接调父类ClassLoader方法就行啦；其实是这样的，我们可以从上面注释看出来，我已经加*标出了，目的是当类加载器失效之后，关闭所有外部的连接，防止内存泄漏以及
长期的资源占用带来的带宽消耗

#### 2.ClassLoader的getResource方法
```
public URL getResource(String name) {
        URL url;
        if (parent != null) {
        		// 这里可以明显的看出来，这里是一个递归调用，当加载器过来查找文件时，会先去当前加载器的parent中去找
            url = parent.getResource(name);
        } else {
        		// 可以知道，这里是最先被执行的，因为bootstrap的父加载器为空，所以就会去JAVA_HOME\jre\lib目录下找是否存在名称是name的文件
            url = getBootstrapResource(name);
        }
        if (url == null) {
        		// 当上面跟加载器未找到该文件时，就会去下面的每个加载器去找直至到当前加载器所加载的内容，需要注意的是，这里都已经存在缓存，不需要实时进行磁盘查找
            url = findResource(name);
        }
        return url;
    }
```
这里parent很有意思，就必须提到JVM中的加载器的父子关系，其实之前的类继承图中也有一点，父子关系图见下图
![](https://zhangxiankui.github.io/imgs/originSource/jdk/ClassLoader-parent-relation.jpg)
下面以一个例子去介绍这个加载过程：
```
// 通过Test类的加载器去获取mybatis的核心配置文件mybatis-config.xml
Test.class.getClassLoader().getResourceAsStream("mybatis-config.xml");
```
- 1.首先Test类是我定义在项目中的，会被编译器编译到target/classes目录下，显而易见其加载器是APP加载器
- 2.当调用getResourceAsStream方法时，URLClassLoader的该方法会被调用，然后会调用getResource方法
- 3.在该方法里面首先是getBootstrapResource(name)被调用，会去JAVA_HOME\jre\lib查找mybatis-config.xml文件，肯定找不到，文件放在类目录下
- 4.此时递归退出到上一层，就会由EXT加载器调用findResource（当然调用的是父类的URLClassLoader方法），会去JAVA_HOME\jre\lib\ext目录下去找，还是找不到
- 5.此时会退出到最外面一层，也就是APP加载器调用findResource，去target/classes目录下找文件名称是mybatis-config.xml的文件，这时候会找到该文件，并返回该文件的输入流
- 本来是想去里面findResource中做深入的分析，但是看了下，里面也就是拿到初始化的URLClassPath列表来进行匹配文件名，来查到是否存在文件，所以意义不大，这里不再扩展

## 四、总结
- 1.以上已经分析了类加载器中如何来进行文件扫描的，对于目前开源的实现项目文件加载的类底层都是调用Classloader的getResourceAsStream方法
- 2.还有同学会说，Class也是可以加载文件，可以获取文件的输入流，其实内部也是调用的Classloader的方法，下面贴一下源码解析一下

```
// 这个方法就是Class类的方法，可以看出就是获取了加载器之后再调用加载器的getResourceAsStream
public InputStream getResourceAsStream(String name) {
				// 这个方法时处理name的文件名的方法，下面做解析
        name = resolveName(name);
        
        ClassLoader cl = getClassLoader0();
        if (cl==null) {
            // A system class.
            return ClassLoader.getSystemResourceAsStream(name);
        }
        return cl.getResourceAsStream(name);
    }


// 1.当name值以“/”开头时，其实默认是找类路径的根目录下查找
// 2.当name值不以“/”开头时，其实就是找当前类的目录下查找该name文件
private String resolveName(String name) {
        if (name == null) {
            return name;
        }
        if (!name.startsWith("/")) {
            Class<?> c = this;
            while (c.isArray()) {
                c = c.getComponentType();
            }
            String baseName = c.getName();
            int index = baseName.lastIndexOf('.');
            if (index != -1) {
                name = baseName.substring(0, index).replace('.', '/')
                    +"/"+name;
            }
        } else {
            name = name.substring(1);
        }
        return name;
    }
```
- 3.有兴趣的话可以去翻一下org.apache.ibatis.io.Resources类加载文件的方法，看是以什么方式加载的？