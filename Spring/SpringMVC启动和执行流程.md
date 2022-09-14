## SpringMVC的启动

spring的启动如下：

1. StandardContext启动时，startInternal()发布一个CONFIGURE_START_EVENT事件，由ContextConfig监听器进行处理
2. ContextConfig调用lifecycleEvent()->configureStart()->webConfig()
3. webConfig()解析web.xml，然后用processServletContainerInitializers方法**扫描项目中**（包括引入的jar）的META-INF\services\javax.servlet.ServletContainerInitializer文件，加载文件中的类。springmvc中为：org.springframework.web.SpringServletContainerInitializer
4. webConfig()在末尾会继续拿到该类上的@HandlesTypes的注解，并查找该注解所设置类型的所有实现类。springmvc设置的为：@HandlesTypes(WebApplicationInitializer.class)
5. 设置完之后，startInternal()在后面会调用SpringServletContainerInitializer的onStartup()方法，该方法先实例化@HandlesTypes设置的WebApplicationInitializer.class类，然后用该类对象设置我们的springmvc容器。该类的父类设置RootConfig，当前类设置WebConfig，并注册DispatcherServlet，设置其启动优先级、映射路径等。所以我们只要继承了WebApplicationInitializer.class类，我们的配置就会生效。
6. 在第五步的设置中，除了加载我们的配置，还向ServletContext（StandardContext的facade）添加了一个 ContextLoaderListener，这个Listener封装了我们的RootConfig.class。然后startInternal()后面会调用listenerStart()，使用该监听器初始化我们的spring容器，初始化spring容器之后，StandardContext会加载初始化每一个servlet，此时我们的DispatcherServlet就会使用配置进行初始化了，初始化的Bean会放在spring容器中。

由前文可知，tomcat最终将请求映射到了一个具体的Servlet上面，所以SpringMVC的DispatcherServlet也是这样进行映射的。

~~~java
// Spring-mvc项目中的SpittrWebAppInitializer类
// 将DispatcherServlet映射到“/”
@Override
protected String[] getServletMappings() {
    return new String[]{"/"};
}
~~~

因为将DispatcherServlet映射到“/”，所以所有的web请求都会由DispatcherServlet进行处理，DispatcherServlet会去寻找相应的Controller进行处理。

### 请求执行过程

![](SpringMVC%E5%90%AF%E5%8A%A8%E5%92%8C%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.assets/Springmvc%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

**注：**

**/  ：**表示（拦截/请求）路径

**/*  ：**表示（拦截/请求）所有文件夹和静态资源。**不包括子文件夹**。

**/\** ：**表式所有的文件夹和静态资源。**包括子文件夹**。



1. 请求过来后，tomcat会通过**/**映射到DispatcherServlet，调用其service方法
2. DispatcherServlet的service方法经过一系列调用，会遍历handlerMappings去寻找请求对应的HandlerMethod，HandlerMethod里面封装了我们请求SpringMVC里面的那个方法和这个方法对应Controller
3. 找到对应的HandlerMethod后，会通过对应的handlerMapping将HandlerMethod与这个mapping对应的interceptor封装成HandlerExecutionChain
4. 拿到HandlerExecutionChain之后，会查找能够适配这种HandlerMethod类型的适配器HandlerAdapter。然后使用参数解析器给对应方法的参数进行注入
5. 最后，首先执行所有interceptor的prehandle，然后通过这个适配器执行对应的方法，（如果是标记了@ResponseBody，那么直接给客户端写回响应）并返回modelAndView，接着执行interceptor的postHandle方法，最后执行interceptor的afterCompletion方法