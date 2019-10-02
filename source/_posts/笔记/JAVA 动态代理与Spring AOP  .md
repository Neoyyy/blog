---
title: JAVA 动态代理与Spring AOP 
date: 2019-06-26 21:02:16
comments: true
categories: 
- 买的课怎么也得学完
tags: 
- java
---

疯狂加班之后终于有两天是早点下班的了,本篇主要讲述以下内容

* 静态代理
* 动态代理
	* jdk动态代理
	* cglib动态代理。
* Spring AOP中是怎么使用到动态代理的
* 在spring的Cglib2AopProxy中使用到的transient关键字是做什么的
<!-- more -->
## 先来说说java的三种代理模式。

代理模式proxy是一种设计模式，假设目标对象A有功能functionA()，只负责他对应的业务逻辑，而调用时想在业务逻辑之前或者之后想打印些系统日志，这时候存在一个对A的增强对象proxyA，扩展了A的功能，使得通过代理对象proxyA访问目标对象A，在目标对象A实现功能的基础上扩展了额外的系统日志打印功能，在不修改A的基础上扩展了目标A的功能并且调用目标对象。
### 静态代理

**静态代理的时候首先需要有一个接口或者父类供代理类和目标类一同实现或继承**。
代码如下
接口**IBussiness.java**  

```java
public interface IBussiness {
        void doSomething();
}

```



目标类**Target.java**  

```java
public class Target implements IBussiness{
    @Override
    	public void doSomething() {
        System.out.println("do something");
    	}
}
```


代理类**TargetProxy.java**  

```java
public class TargetProxy implements IBussiness{
	private IBussiness target;

	public TargetProxy(IBussiness target) {
    this.target = target;
	}

	@Override
	public void doSomething() {
    System.out.println("before do something");
    target.doSomething();
    System.out.println("after do something");
	}
}
```


测试类**AppTest.java**  
```java
public class AppTest {
    /**
     * Rigorous Test :-)
     */
    @Test
    public void shouldAnswerWithTrue()
    {
        assertTrue( true );
        Target target = new Target();
    
        TargetProxy proxy = new TargetProxy(target);
    
        proxy.doSomething();
    }
}
```

运行结果
![静态代理图1](/images/imageForPost/笔记/动态代理/静态代理图1.png)

------

​		可见代理可以在不修改目标对象的情况下扩展对应的功能，但是代理对象和目标对象要实现相同的接口，如果业务中存在大量需要被代理的类则会增加很多不必要的维护工作。
​		为了解决这一问题，可以看下jdk动态代理。
​		动态代理不要求代理类与目标类实现相同的接口，但是目标类要求实现接口，通过在运行时创建实现了指定接口的对象来实现目标对象的扩展。
### JDK动态代理
主要使用到**java.lang.reflect**的**Proxy**类的**newProxyInstance**方法 

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)throws IllegalArgumentException
```

主要有三个参数 
* ClassLoader loader  

  ClassLoader 类加载器，用来加载生成的类，类加载器可以参考这篇

  [这篇]: 

  

* Class<?>[] interfaces

	interfaces  代理类实现的被代理类的接口。

* InvocationHandler h  
	InvocationHandler 代理类的扩展处理器，具体扩展逻辑在该处理器中实现。
------
继续使用上面用到的**IBussiness.java** 接口
代理处理器**ProxyInvocationHandler.java**

```java
public class ProxyInvocationHandler implements  InvocationHandler {
        
    	//被代理的目标对象
      private Object target;
        
      public ProxyInvocationHandler(Object target) {
        this.target = target;
      }
        
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      System.out.println("before do something");
      //通过反射调用目标对象的方法，如果invoke传入代理对象则会出现递归调用代理对象的代理方法
      Object result = method.invoke(target, args);
      System.out.println("after do something");
      return result;
      }


      public Object getProxy(){
        return Proxy.newProxyInstance(this.getClass().getClassLoader(),	target.getClass().getInterfaces(), this);
    
      }
}

```

测试类**JdkProxy.java**
```java
public class JdkProxy {
	@Test
	public void shouldAnswerWithTrue(){
       IBussiness target = new Target();
    
       IBussiness proxyObj = (IBussiness) new ProxyInvocationHandler(target).getProxy();
    
       proxyObj.doSomething();
	}
}
```
执行效果

![jdk代理图1](/images/imageForPost/笔记/动态代理/jdk代理图1.png)

------



### cglib的代理
	与jdk代理不同的是cglib代理是通过运行时通过字节码库生成目标对象的子类，所以不需要像jdk代理一样目标对象需要实现接口，Spring AOP中就是根据目标对象是否实现了接口来确定使用jdk代理还是cglib代理，具体的选择在后面会讲。

测试工程使用的是**maven**，所以在工程**pom**中添加cglib的依赖
```shell
    <dependency>
      <groupId>cglib</groupId>
      <artifactId>cglib</artifactId>
      <version>3.1</version>
    </dependency>
```

继续使用上面用到的**IBussiness.java** 接口
代理处理器**CglibProxy.java**

```java
public class CglibProxy implements MethodInterceptor {
    
    //目标对象
    private Object target;
    
    public CglibProxy(Object target) {
        this.target = target;
    }
    
    public Object getProxyObj(){
        //cglib工具类
        Enhancer en = new Enhancer();
        //设置被代理对象
        en.setSuperclass(target.getClass());
        //设置扩展处理器
        en.setCallback(this);
        //返回代理对象
        return en.create();
    }
    
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("before do something by cglib");
        Object result = method.invoke(target, args);
        System.out.println("after do something by cglib");
        return result;
    }
}
```
测试类**CglibProxyTest.java**
```java
public class CglibProxyTest {
    @Test
    public void shouldAnswerWithTrue()
    {
        IBussiness target = new Target();
    
        IBussiness proxyObj = (IBussiness) new CglibProxy(target).getProxyObj();
    
        proxyObj.doSomething();


    }
}
```

运行结果

![cglib代理图1](/images/imageForPost/笔记/动态代理/cglib代理图1.png)

可见cglib和jdk代理在代码结构上非常类似，Spring AOP则是使用这两种方式创建代理，Spring AOP中代理的扩展方法即advice扩展方法是另外指定的，而前面的代理扩展则是写死的，如何使得代理类能使用我们指定的方法呢，像AOP一样可以使用前置通知、后置通知和环绕通知。最简单的方法就是给**ProxyInvocationHandler**的构造方法传入想要用来扩展的方法，然后在目标对象方法的invoke之前调用扩展方法即可。
接下来简单的看下Spring的AOP
手边的工程是用的Spring 3.0.5版本
在**org.springframework.aop.framework**包下的**DefaultAopProxyFactory**类中有个**public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException** 方法

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
            return new JdkDynamicAopProxy(config);
        } else {
            Class targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
            } else if (targetClass.isInterface()) {
                return new JdkDynamicAopProxy(config);
            } else if (!cglibAvailable) {
                throw new AopConfigException("Cannot proxy target class because CGLIB2 is not available. Add CGLIB to the class path or specify proxy interfaces.");
            } else {
                return DefaultAopProxyFactory.CglibProxyFactory.createCglibProxy(config);
            }
        }
}
```
会根据被代理对象是否有实现接口来选择使用jdk代理还是cglib代理
跟进**JdkDynamicAopProxy**类可以看到**getProxy()**方法通过**Proxy.newProxyInstance**返回了一个代理对象

```java
public Object getProxy(ClassLoader classLoader) {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
        }
    
        Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
        this.findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```


跟进,则可以看到对应的**getProxy()**方法
```java
public Object getProxy(ClassLoader classLoader) {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating CGLIB2 proxy: target source is " + this.advised.getTargetSource());
        }
    
        try {
            Class rootClass = this.advised.getTargetClass();
            Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");
            Class proxySuperClass = rootClass;
            int x;
            if (AopUtils.isCglibProxyClass(rootClass)) {
                proxySuperClass = rootClass.getSuperclass();
                Class[] additionalInterfaces = rootClass.getInterfaces();
                Class[] var8 = additionalInterfaces;
                x = additionalInterfaces.length;
    
                for(int var6 = 0; var6 < x; ++var6) {
                    Class additionalInterface = var8[var6];
                    this.advised.addInterface(additionalInterface);
                }
            }
    
            this.validateClassIfNecessary(proxySuperClass);
            Enhancer enhancer = this.createEnhancer();
            if (classLoader != null) {
                enhancer.setClassLoader(classLoader);
                if (classLoader instanceof SmartClassLoader && ((SmartClassLoader)classLoader).isClassReloadable(proxySuperClass)) {
                    enhancer.setUseCache(false);
                }
            }
    
            enhancer.setSuperclass(proxySuperClass);
            enhancer.setStrategy(new UndeclaredThrowableStrategy(UndeclaredThrowableException.class));
            enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
            enhancer.setInterceptDuringConstruction(false);
            Callback[] callbacks = this.getCallbacks(rootClass);
            enhancer.setCallbacks(callbacks);
            enhancer.setCallbackFilter(new Cglib2AopProxy.ProxyCallbackFilter(this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
            Class[] types = new Class[callbacks.length];
    
            for(x = 0; x < types.length; ++x) {
                types[x] = callbacks[x].getClass();
            }
    
            enhancer.setCallbackTypes(types);
            Object proxy;
            if (this.constructorArgs != null) {
                proxy = enhancer.create(this.constructorArgTypes, this.constructorArgs);
            } else {
                proxy = enhancer.create();
            }
    
            return proxy;
        } catch (CodeGenerationException var9) {
            throw new AopConfigException("Could not generate CGLIB subclass of class [" + this.advised.getTargetClass() + "]: " + "Common causes of this problem include using a final class or a non-visible class", var9);
        } catch (IllegalArgumentException var10) {
            throw new AopConfigException("Could not generate CGLIB subclass of class [" + this.advised.getTargetClass() + "]: " + "Common causes of this problem include using a final class or a non-visible class", var10);
        } catch (Exception var11) {
            throw new AopConfigException("Unexpected AOP exception", var11);
        }
}
```

在**Cglib2AopProxy**的属性中有一个transient关键字，之前基本没看到过（果然是我见识少啊）
这也是java的关键字之一
这个关键字用来标示某个属性不被序列化
定义一个会被序列化的类**TrabsientTest.java**

```java
public class TransientTest implements Serializable {
    
    private String field1;
    
    private transient String field2;


    public TransientTest(String field1, String field2) {
        this.field1 = field1;
        this.field2 = field2;
    }
    public TransientTest() {
        this.field2 = "???";
    
    }
    @Override
    public String toString() {
        return "TransientTest{" +
                "field1='" + field1 + '\'' +
                ", field2='" + field2 + '\'' +
                '}';
    }
}
```

然后创建一个该类的对象并且序列化它 **TestSerial.java**
```java
public class TestSerial {
    private TransientTest obj;
    
    @Test
    public void shouldAnswerWithTrue()
    {


        TransientTest obj = new TransientTest("test1","test2");
    
        System.out.println(obj.toString());
    
        try {
            ObjectOutputStream o = new ObjectOutputStream(new FileOutputStream("obj"));
            o.writeObject(obj);
            o.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }


        try {
            ObjectInputStream in =new ObjectInputStream(new FileInputStream("obj"));
            TransientTest logInfoIn = (TransientTest)in.readObject();
            System.out.println(logInfoIn.toString());
        } catch(Exception e) {
            e.printStackTrace();
        }
    
    }
}
```
结果如图

![transient图1](/images/imageForPost/笔记/动态代理/transient图1.png)

​	执行后可以看到field2并没有值，在序列化时略过了field2，在反序列化的时候并没有执行构造函数给field2赋值“？？？”，反序列化并不会通过构造函数进行创建对象，而是载入了该类对象的持久化状态。
