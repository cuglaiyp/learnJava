# 以JDBC为例谈双亲委派模型的破坏

java本身有一套资源管理服务JNDI，是放置在rt.jar中，由启动类加载器加载的。以对数据库管理JDBC为例，
java给数据库操作提供了一个Driver接口：

```java
public interface Driver {

   
    Connection connect(String url, java.util.Properties info)
        throws SQLException;
    boolean acceptsURL(String url) throws SQLException;
    DriverPropertyInfo[] getPropertyInfo(String url, java.util.Properties info)
                         throws SQLException;
    int getMajorVersion();
    int getMinorVersion();
    boolean jdbcCompliant();
    public Logger getParentLogger() throws SQLFeatureNotSupportedException;
}
```

然后提供了一个DriverManager来管理这些Driver的具体实现：

```java
public class DriverManager {


    // List of registered JDBC drivers 这里用来保存所有Driver的具体实现
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
    public static synchronized void registerDriver(java.sql.Driver driver)
        throws SQLException {

        registerDriver(driver, null);
    }

    public static synchronized void registerDriver(java.sql.Driver driver,
            DriverAction da)
        throws SQLException {

        /* Register the driver if it has not already been added to our list */
        if(driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }
    

}
```

这里省略了大部分代码，可以看到我们使用数据库驱动前必须先要在DriverManager中使用registerDriver()注册，然后我们才能正常使用。

## 不破坏双亲委派模型的情况（不使用JNDI服务）

我们看下mysql的驱动是如何被加载的：

```cpp
// 1.加载数据访问驱动
Class.forName("com.mysql.jdbc.Driver");
//2.连接到数据"库"上去
Connection conn= DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb?characterEncoding=GBK", "root", "");
```

**为什么说这里没有破坏双亲委派：因为这里的`Class.forName`会拿到caller的加载器进行Driver的加载**

核心就是这句Class.forName()触发了mysql驱动的加载，我们看下mysql对Driver接口的实现：

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

可以看到，Class.forName()其实触发了静态代码块，然后向DriverManager中注册了一个mysql的Driver实现。
这个时候，我们通过DriverManager去获取connection的时候只要遍历当前所有Driver实现，然后选择一个建立连接就可以了。

## 破坏双亲委派模型的情况

在JDBC4.0以后，开始支持使用spi的方式来注册这个Driver，具体做法就是在mysql的jar包中的META-INF/services/java.sql.Driver 文件中指明当前使用的Driver是哪个，然后使用的时候就直接这样就可以了：

```bash
 Connection conn= DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb?characterEncoding=GBK", "root", "");
```

可以看到这里直接获取连接，省去了上面的Class.forName()注册过程。
现在，我们分析下看使用了这种spi服务的模式原本的过程是怎样的:

- 第一，从META-INF/services/java.sql.Driver文件中获取具体的实现类名“com.mysql.jdbc.Driver”
- 第二，加载这个类，这里肯定只能用class.forName("com.mysql.jdbc.Driver")来加载

**好了，问题来了，Class.forName()加载用的是调用者的Classloader，这个调用者DriverManager是在rt.jar中的，ClassLoader是启动类加载器，而com.mysql.jdbc.Driver肯定不在<JAVA_HOME>/lib下，所以肯定是无法加载mysql中的这个类的。这就是双亲委派模型的局限性了，父级加载器无法加载子级类加载器路径中的类。**

那么，这个问题如何解决呢？按照目前情况来分析，这个mysql的drvier只有应用类加载器能加载，那么我们只要在启动类加载器中有方法获取应用程序类加载器，然后通过它去加载就可以了。这就是所谓的线程上下文加载器。
**线程上下文类加载器可以通过Thread.setContextClassLoaser()方法设置，如果不特殊设置会从父类继承，一般默认使用的是应用程序类加载器**

**很明显，线程上下文类加载器让父级类加载器能通过调用子级类加载器来加载类，这打破了双亲委派模型的原则**

现在我们看下DriverManager是如何使用线程上下文类加载器去加载第三方jar包中的Driver类的。

```java
public class DriverManager {
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    private static void loadInitialDrivers() {
        //省略代码
        //这里就是查找各个sql厂商在自己的jar包中通过spi注册的驱动
        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
        Iterator<Driver> driversIterator = loadedDrivers.iterator();
        try{
             while(driversIterator.hasNext()) {
                driversIterator.next();
             }
        } catch(Throwable t) {
                // Do nothing
        }

        //省略代码
    }
}
```

使用时，我们直接调用DriverManager.getConn()方法自然会触发静态代码块的执行，开始加载驱动
然后我们看下ServiceLoader.load()的具体实现：

```php
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader){
    return new ServiceLoader<>(service, loader);
}
```

可以看到核心就是拿到线程上下文类加载器，然后构造了一个ServiceLoader,后续的具体查找过程，我们不再深入分析，这里只要知道这个ServiceLoader已经拿到了线程上下文类加载器即可。
接下来，DriverManager的loadInitialDrivers()方法中有一句**driversIterator.next();**,它的具体实现如下：

```csharp
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        //此处的cn就是产商在META-INF/services/java.sql.Driver文件中注册的Driver具体实现类的名称
        //此处的loader就是之前构造ServiceLoader时传进去的线程上下文类加载器
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    //省略部分代码
}
```

现在，我们成功的做到了通过线程上下文类加载器拿到了应用程序类加载器（或者自定义的然后塞到线程上下文中的），同时我们也查找到了厂商在子级的jar包中注册的驱动具体实现类名，这样我们就可以成功的在rt.jar包中的DriverManager中成功的加载了放在第三方应用程序包中的类了。

## 总结

这个时候我们再看下整个mysql的驱动加载过程:

- 第一，获取线程上下文类加载器，从而也就获得了应用程序类加载器（也可能是自定义的类加载器）
- 第二，从META-INF/services/java.sql.Driver文件中获取具体的实现类名“com.mysql.jdbc.Driver”
- 第三，通过线程上下文类加载器去加载这个Driver类，从而避开了双亲委派模型的弊端

很明显，mysql驱动采用的这种spi服务确确实实是破坏了双亲委派模型的，毕竟做到了父级类加载器加载了子级路径中的类。