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
            // 调用父类中，也就是Proxy中保存的invoke方法去执行我们的被代理方法。
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

