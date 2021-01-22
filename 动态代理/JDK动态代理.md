# JDK动态代理

jdk动态代理，思路、代码上还是很好理解的。我们先来看一段，jdk动态代理的使用

~~~java
public class Main {
    // main方法
    public static void main(String[] args) {
        // 这句话是把动态代理过程生成的代理类字节码保存成class文件
        // System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 生成一个需要被代理的对象
        SmsService service = new SmsServiceImpl();
        // 调用句柄。意为，这个类会去增强并执行代理方法。既然要增强原方法，所以需要把原方法传进去，也就是service
        InvocationHandler handler = new MyInvocationHandler(service);
        // 通过newProxyInstance方法，得到一个代理类。
        SmsService proxyService = (SmsService) Proxy.newProxyInstance(service.getClass().getClassLoader(), service.getClass().getInterfaces() , handler);
        // 代理类实现了被代理类所实现的接口，所以它也实现了该接口所需要实现的所有方法。也就是说，代理类增强了接口里面的所有方法。
        proxyService.send("java");
    }
}

// jdk动态代理只能代理有接口的实现类
interface SmsService{
    void send(String s);
}

// 实现一个接口
class SmsServiceImpl implements SmsService{

    // 定义好方法里面我们的业务逻辑
    @Override
    public void send(String s) {
        System.out.println("发送短信");
    }
}

// 这个类是最终增强并执行方法的类调用句柄类
class MyInvocationHandler implements InvocationHandler{

    // 保存需要被代理类
    private SmsService service;

    public MyInvocationHandler(SmsService service){
        this.service = service;
    }

    /***
     * 这个方法就是用来增强并执行被代理类方法的方法
     * @param proxy 生成的代理对象本身。通过这点，可以看出，这个invoke方法，应该就是代理对象调用的，并传了this
     * @param method 被代理的方法
     * @param args 被代理方法执行所需要的参数
     * @return
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 被代理方法执行之前的增强
        System.out.println("代理前");
        // 被代理方法的执行
        method.invoke(service, args);
        // 被代理方法执行之后的增强
        System.out.println("代理后");
        return null;
    }
}
~~~

我们来看一下`proxyService.send(“java”)`这句话怎么调用到`MyInvocationHandler`。我们来看一下代理过程生成的代理类反编译后的文件

~~~java
final class $Proxy0 extends Proxy implements SmsService {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    // 可以看到，代理类中，实现了需要被代理接口中的方法
    public final void send(String var1) throws  {
        try {
            // 调用父类中，也就是Proxy中保存InvocationHandler的invoke方法去执行我们的被代理方法。
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    static {
        try {
            // 4个方法，都是通过反射获得的，其中就有我们接口中的方法m3
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("main.SmsService").getMethod("send", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
~~~

直观地看一下调用过程

​                                                                                                                                                |——>`增强代码`       

`$Proxy.send(args)`——>`MyInvocationHandler.invoke(method(send), args)`|——>`method.invoke(SmsServiceImpl,args)`

​                                                                                                                                                |                    ||

​                                                                                                                                                |          `SmsServiceImpl.send(args)` 

​                                                                                                                                                |——>`增强代码`

**大白话来说就是这样**：

1. 兄弟（目标类）你把你实现接口告诉我（代理类），这样我去实现这个接口，一来知道你里面的方法，二来我们俩的类型就是一样的了，这样的话在别人需要你的时候，把我给人家，方法我也有、类型也一样，达到以假乱真的效果。（`newProxyInstance`第二个参数的意义）
2. 但是，虽然说我有你的同名方法，在这个方法中可以直接去调用你的同名方法，达到代理你方法的效果，但是有两个问题：一是我并不知道怎么去增强你方法，二是我也不知道在有了增强代码后什么时机去调用你的同名方法。所以我们俩需要约定一下，你在`InvocationHandler`类型的`invoke`方法中写好怎么增强、增强的时候什么时机调用的你的方法，然后把这玩意儿给我保存。当别人调我方法的时候，我调这个`invoke`方法。（`newProxyInstance`第三个参数的意义）
3. 最后还有一个点就是，因为我的是运行时生成的字节码，我要用类加载器把自己加载进jvm里面，为了保证咱俩在jvm层面也是同样的类型，你需要把你的类加载器也传递给我。（`newProxyInstance`第一个参数的意义）