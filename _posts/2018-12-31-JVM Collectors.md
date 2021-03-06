---
layout: post
title:  "JVM内存回收器"
categories: JVM GC
tags:  GC 内存回收器
author: zhangxiankui
---

* content
{:toc}


## 一、GC和回收器简介
- JVM是通过内存回收器来自动完成内存回收的，也就是通常所说的GC
- Hotspot虚拟机内存回收器分为新生代和老年代，这是由于该虚拟机采用分代收集算法（新生代、老年代相关的JVM运行时内存后期会单独分享）
- 新生代回收器有serial，parall，parallel scavenge三种
- 老年代收集器有serial old，parallel old，CMS，G1
	
## 二、回收器特性
- serial和serial old分别是采用复制清除算法的新生代和老年代回收器，收集器运作时会导致用户线程停止（虚拟机通过一个标志位来实现的，当收集线程执行时，将标志位置为0）
- paral和parallel old其实就是serial和serial old的多线程版本
- paraller scavenge是采用复制清除算法的吞吐量优先的内存回收器，所谓吞吐量，就是指用户线程执行时间除以用户线程和收集线程总和的比例，一般应用在后台运行的程序上
- CMS(concurrent mark sweep)回收器是基于标记清除算法的并发老年代收集器，它的处理分为四个阶段，初始标记，并发标记，最终标记，并发清理。初始标记就是在标记和根节点有关系的对象，
并发标记是根节点跟踪，标记点跟踪的过程，在初始标记和最终标记中其实还是会导致用户线程的停止，但是这比并发标记和清除所消耗时间小的多，这也是CMS为什么能够达到快速GC，低延时的效果，适合前端交互系统
- G1是1.7提出的高性能回收器，也是基于低延时的回收器，采用标记清除算法，内存回收步骤和CMS差别不大（另：内存回收涉及的回收算法可自行百度，后期会单独分享）
		
## 三、回收器的优缺点分析
- CMS收集器的缺点，由于CMS采用标记清除算法来进行内存回收，所以会有内存碎片的存在，会导致某些大对象无法分配内存而导致内存回收，这样就会导致GC次数增加；
另外由于标记是和用户线程并发执行的，所以会存在浮动垃圾，所谓浮动垃圾，意思是指这次内存回收期间产生的无引用对象（内存垃圾），但是未在本次被标记，所以所占用的内存空间并不会回收
- G1收集器和CMS的对比，对于GC的时间是可控的，这是由于G1收集器采用分治策略，将内存分为多个segement，然后当发生GC时，就判断哪个segment触发内存回收会获得最大的效率，并且减小GC带来的停顿时间

## 四、总结
- 在系统选择内存回收器的时，需要根据业务特征选用对应的回收器，正如上文所说，当系统是一个面向用户系统时，为了给用户更好的使用体验，需要使用老年代可以使用
延迟低的CMS或者G1回收器，新生代的话就可以使用parallel old，这样可以降低系统GC带来的延时
- 当系统是一个后台系统，例如某数据系统，只是在后台去处理数据，我们希望海量数据能够快速的处理，这个时候就可以选用paraller scavenge来作为内存回收器
- 这里只是简单介绍每个回收器的特性，让自己对回收器有个大概的了解                                
