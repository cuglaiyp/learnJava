### Spring MVC的配置思路

![image-20201201192435721](Spring-mvc.assets/image-20201201192435721.png)

1. 容器启动的时候会在类路径中查找实现ServletContainerInitializer接口的类。如果找到，就会用这个实现类配置Servlet的容器
2. Spring为我们提供了这个实现类：SpringServletContainerInitializer。
   但是这个Spring并没有用这个类来配置Servlet容器，而是查找WebApplicationInitializer的实现类
   Spring给WebApplicationInitializer也做了一个基础实现类，也就是AbstractAnnotationConfigDispatcherServletInitializer
3. 我们自己定义类SpittrWebAppInitializer就会被查找，用来配置Servlet容器

### bug解决

##### bug1：

按照Spring实战4.0用纯注解搭建的Spring MVC一直报404的错误。

##### 解决：

打开Project Structure，在如下位置即可解决

![image-20201202113805449](Spring-mvc.assets/image-20201202113805449.png)

##### bug2

spring-mvc项目配置的jpa与spring-jpa项目的配置一模一样，可spring-mvc项目跑不起来。

##### 解决

玄学解决，将Artifact删掉重新配置一遍，重新命一个新名字试试。详情见spring-mvc项目，具体配置都有注释。