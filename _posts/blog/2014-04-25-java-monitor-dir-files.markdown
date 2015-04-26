---
layout: post
title: "使用Java监控操作系统目录或文件变化"
date: 2013-04-25 16:54:46
author: Admin
categories:
- blog
- java
- java 7
---

####如何监测目录或文件变化？

监听文件更新在实际项目开发中比较常见，一般常规的做法就是定时的去查看文件或目录是否发生变化，即采用轮询的思路。但这样会产生的问题就是如果目录或文件根本没有发生任何变化，则将大大消耗系统的资源。为了避免不必要的开销，必须调用底层系统的API来完成我们的需求。<!--more-->

但Java调用底层系统API必须使用JNI的方式，那是不是已经有开源的库实现了我们的需求呢？答案是肯定的，我们没有必要重复的去造轮子，奉行拿来主义即可。[jpathwatch]是一个监测目录或文件改变的Java库，它使用本地系统平台的函数来完成，避免不必要的轮询。该库主要针对目录的如下事件进行监听：

* 文件创建和删除
* 文件修改
* 文件重命名
* 修改子目录（采用递递归监测方式）
* 非法操作（监测的目录不可达）

它目前所支持的平台如下：

* Windows (Windows 2000, XP, Vista, 7, 32bit/64bit)
* Linux (x86, 32bit/64bit)
* Mac OS X　(x86, 32bit/64bit, tested on 10.5)　(PPC, tested on 10.4)
* FreeBSD (x86, 32bit)

JDK最低要求为5，从整体上看该库具有高可用性，但实际开发时经常出现不可预料的问题，可能是系统资源不够或平台的兼容性等，崩溃性事件经常发生。

####难道Java就没有内置的watch库吗？

其实很多的库开始时是开源的，然后Java开发者看到这个库不错，就集成到标准JDK中，该库也是如此。Java7 java.nio.file库提供了文件改变通知API即Watch Service API。此API可用Watch Service注册一个或多个目录，然后通知一个文件创建、删除、修改事件给注册process。注册process使用一个线程去监测并处理任何已注册的文件事件。基本步骤如下：

* 针对需监听文件系统创建WatchService“watcher”；
* 对每个目录注册到watcher，注册时指定所通知事件。对于每个目录将获得一个WatchKey实例；
* 实现一个无限循环等待事件发生，当事件发生时，注册的key触发并放置到watcher队列中；
* 从watcher队列获取key（可从key中获得文件名字）；
* 对于key获取每一个等待事件和处理流程；
* 重置key去等待文件事件；
* 关闭service。当线程退出或关闭（closed方法），watch service退出。

示例代码如下：

1. watch函数
<pre><code>
    public void watch() {
      WatchService watchService = FileSystems.getDefault().newWatchService();
      name.pachler.nio.file.Path wachedPath = Paths.get(aDirPath);
      try {
        WatchKey key = wachedPath.register(watchService,
          StandardWatchEventKind.ENTRY_CREATE, 
          StandardWatchEventKind.ENTRY_DELETE,
          StandardWatchEventKind.ENTRY_MODIFY);
      } catch (UnsupportedOperationException uox) {
      } catch (IOException iox) { }
      Thread thread = new Thread() {
        @Override
        public void run() {
          try {
        processEvents();
          } catch (IOException e) { }
        }
      };
      thread.start();
    }
</code></pre>

2. 处理事件函数
<pre><code>
    public void processEvents() {
      while(true){
        WatchKey signalledKey; //wait for key to be signaled
        try {
          signalledKey = watchService.take();
        } catch (InterruptedException ix){
          continue; //ignore being interrupted
        } catch (ClosedWatchServiceException cwse){
          break; // other thread closed watch service
        }
        // get watch events of events from key
        List<WatchEvent<?>> watchEvents = signalledKey.pollEvents();
        // VERY IMPORTANT! call reset() AFTER pollEvents() to allow the
        // key to be reported again by the watch service
        signalledKey.reset();
        for(WatchEvent event : watchEvents){
          name.pachler.nio.file.Path context = (name.pachler.nio.file.Path)event.context();
          String enventFileName = context.toString();
          if(enventFileName.equals("watch_file_name")) {
            if(event.kind() == StandardWatchEventKind.ENTRY_CREATE) {
            } else if(event.kind() == StandardWatchEventKind.ENTRY_DELETE) {
            } else if (event.kind() == StandardWatchEventKind.ENTRY_MODIFY) {
            } else if(event.kind() == StandardWatchEventKind.OVERFLOW) {
            }
          }
        }
      }
    }
</code></pre>

[jpathwatch]: https://jpathwatch.wordpress.com/
