StandardService启动时，会调用mapperListener.start()->startInternal()，该方法会将StandardService中的Mapper进行如下的初始化：

1. 首先registerHost()，将Engin中的defaulthost封装成MappedHost，存入mapper

2. 接着registerContext()，将该host中的context容器封装成MappedContext存入1中MappedHost。具体过程如下

   1. 用prepareWrapperMappingInfo()方法将wrapper的**mapping映射路径**、**映射路径是否是通配符（wildcard**、**该servlet是否是resourceOnlyServlet**以及**wrapper本身**封装成**WrapperMappingInfo**，放进wrappers集合中。

   2. 包装好之后会用addContextVersion()方法生成ContextVersion。ContextVersion实际上就是StandardContext一个包装类，会存放如下东西：

      ```java
      public final String path;     // context的路径
      public final int slashCount;  // 路径中 / 的数量
      public final WebResourceRoot resources; // 一些资源吧
      public String[] welcomeResources;       // 欢迎页面
      public MappedWrapper defaultWrapper = null; // 默认wrapper，也就是wrapper映射路径为/的wrapper
      public MappedWrapper[] exactWrappers = new MappedWrapper[0]; // wrapper映射路径正常的wrapper，例如/test
      public MappedWrapper[] wildcardWrappers = new MappedWrapper[0]; // wrapper映射路径由 /*结尾的wrapper
      public MappedWrapper[] extensionWrappers = new MappedWrapper[0]; // wrapper映射路径为 *.的wrapper
      public int nesting = 0;
      private volatile boolean paused;
      //父类的属性
      public final String name; // 名字，也就是version
      public final T object; // 存放Context实例
      ```

      可以看到只存一个默认的wrapper，所以当有几个映射路径为”/“的servlet，只有一个会生效

   3. addContextVersion()方法会先new一个ContextVersion并且将name、context、path、slashCount设置进去。

   4. 第3步做好之后，会用addWrappers()方法将这些WrapperMappingInfo生成MappedWrapper。**MappedWrapper看了一下源码和WrapperMappingInfo并没有什么区别，虽然是两个类，但是都只包括1中的那4项**。不同的是，在这个处理过程中，会根据WrapperMappingInfo中wrapper的映射路径，**将路径处理后**，再把MappedWrapper放置在ContextVersion不同的集合中。

   5. 最后将ContextVersion和它的path一起封装成MappedContext放到Mapper中的MappedHost中去。

可以看到郑整个过程，就是通过一层一层扫描我们的容器，将它们的映射信息以及其他的一些相关信息和容器本身封装起来，存入mapper之中。所以Mapper结构也如容器结构类似，host包着context，context包着wrapper。只不过Mapper中的容器与他们之间的映射路径已经强关联在一起了

