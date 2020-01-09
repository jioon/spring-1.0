spring-aop

从最基本的使用开始。

```
public interface ITestBean {

    int getAge();

    void setAge(int age);

    String getName();

    void setName(String name);
}

public class TestBean implements  ITestBean {
	private int age;

	private String name;

	public TestBean() {
	}

	public TestBean(String name, int age) {
		this.name = name;
		this.age = age;
	}

	@Override
	public int getAge() {
		return age;
	}

	@Override
	public void setAge(int age) {
		this.age = age;
	}

	@Override
	public String getName() {
		return name;
	}

	@Override
	public void setName(String name) {
		this.name = name;
	}
}


import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
public class NopInterceptor implements MethodInterceptor {
    private int count;

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        ++count;
        System.out.println("Debug interceptor: count=" + count +
                " invocation=[" + invocation + "]");
        Object rval = invocation.proceed();
        System.out.println("Debug interceptor: next returned");
        return rval;
    }
}
```

beans.xml为

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!-- Simple target -->
    <bean id="test" class="com.youxiao.doc.aop.TestBean">
        <property name="name">
            <value>custom</value>
        </property>
        <property name="age">
            <value>666</value>
        </property>
    </bean>

    <bean id="debugInterceptor" class="com.youxiao.doc.aop.NopInterceptor">
    </bean>

    <bean id="test1"
          class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces">
            <value>com.youxiao.doc.aop.ITestBean</value>
        </property>

        <property name="target">
            <ref local="test"/>
        </property>
        <property name="interceptorNames">
            <value>debugInterceptor</value>
        </property>
    </bean>
</beans>
```

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import java.lang.reflect.Proxy;

public class AopMain {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        ITestBean test1 = (ITestBean) context.getBean("test1");
        System.out.println(test1.getClass());
        System.out.println(Proxy.isProxyClass(test1.getClass()));
        System.out.println(test1.getName());
        System.out.println(test1.getAge());
        System.out.println(test1.returnsThis());
    }
}
```

输出为

class com.sun.proxy.$Proxy3
true
Debug interceptor: count=1 invocation=[Invocation: method=[public abstract java.lang.String com.youxiao.doc.aop.ITestBean.getName()] args=null] target is of class com.youxiao.doc.aop.TestBean]
Debug interceptor: next returned
custom
Debug interceptor: count=2 invocation=[Invocation: method=[public abstract int com.youxiao.doc.aop.ITestBean.getAge()] args=null] target is of class com.youxiao.doc.aop.TestBean]
Debug interceptor: next returned
666

从执行结果来看，MethodInterceptor对接口方法已经起增强作用。以下我们将看下 Spring AOP 的具体实现。

从上面的示例可以看出 Spring AOP 的配置主要基于类 `ProxyFactoryBean` ，那么我们就以此为入口去剖析其实现。

![ProxyFactory](https://github.com/jioon/spring-1.0/blob/master/aop/image/ProxyFactoryBean.jpg?raw=true)

从其继承结构来看，`test1`是factory  bean，且实现了`BeanFactoryAware`接口，意味着getBean访问的是`test1`的getObject方法，且bean创建的过程setBeanFactory方法会被调用。先从`ProxyFactoryBean `的setBeanFactory开始

```java
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
    // 创建拦截链
    createAdvisorChain();
    if (singleton) {
        // 若是单例,提前创建
        // Eagerly initialize the shared singleton instance
        getSingletonInstance();
        // We must listen to superclass advice change events to recache singleton
        // instance if necessary
        addListener(this);
    }
}

private void createAdvisorChain() throws AopConfigException, BeansException {
    if (this.interceptorNames == null || this.interceptorNames.length == 0) {
        //throw new AopConfigException("Interceptor names are required");
        return;
    }

    // Globals can't be last
    if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX)) {
        throw new AopConfigException("Target required after globals");
    }

    // Materialize interceptor chain from bean names
    for (int i = 0; i < this.interceptorNames.length; i++) {
        String name = this.interceptorNames[i];

        // 探查 interceptor name 是否以 * 结尾
        if (name.endsWith(GLOBAL_SUFFIX)) {
            if (!(this.beanFactory instanceof ListableBeanFactory)) {
                throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
            }
            else {
                addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
                                 name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
            }
        }
        else {
            // 获取interceptor bean
            // add a named interceptor
            Object advice = this.beanFactory.getBean(this.interceptorNames[i]);
            //添加advisor
            addAdvisor(advice, this.interceptorNames[i]);
        }
    }
}

private void addAdvisor(Object next, String name) {
    // 查找 advice 匹配的 pointcut, 并创建一个 advisor
    // We need to add a method pointcut so that our source reference matches
    // what we find from superclass interceptors.
    Object advisor = namedBeanToAdvisorOrTargetSource(next);
    if (advisor instanceof Advisor) {
        // if it wasn't just updating the TargetSource
        addAdvisor((Advisor) advisor);
        // Record the pointcut as descended from the given bean name.
        // This allows us to refresh the interceptor list, which we'll need to
        // do if we have to create a new prototype instance. Otherwise the new
        // prototype instance wouldn't be truly independent, because it might
        // reference the original instances of prototype interceptors.
        this.sourceMap.put(advisor, name);
    }
    else {
        // 如果interceptor不是advisor,意味着会被当做代理对象使用,将后续例子
        setTargetSource((TargetSource) advisor);
        // save target name
        this.targetName = name;
    }
}

private Object namedBeanToAdvisorOrTargetSource(Object next) {
    try {
        Advisor adv = GlobalAdvisorAdapterRegistry.getInstance().wrap(next);
        return adv;
    }
    catch (UnknownAdviceTypeException ex) {
        // TODO consider checking that it's the last in the list?
        if (next instanceof TargetSource) {
            return (TargetSource) next;
        }
        else {
            // It's not a pointcut or interceptor.
            // It's a bean that needs an invoker around it.
            return new SingletonTargetSource(next);
        }
    }
}
```

GlobalAdvisorAdapterRegistry是advisor的承载类，通过单例模式实现。其包装方法wrap会将advise包装成advisor对象（DefaultPointcutAdvisor）。

```java
public class GlobalAdvisorAdapterRegistry extends DefaultAdvisorAdapterRegistry {
	private static GlobalAdvisorAdapterRegistry instance = new GlobalAdvisorAdapterRegistry();
	
	public static GlobalAdvisorAdapterRegistry getInstance() {
		return instance;
	}

	private GlobalAdvisorAdapterRegistry() {
	}	
}

public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry {
	
	private List adapters = new LinkedList();
	
	public DefaultAdvisorAdapterRegistry() {
        // 意味着1.0只支持BeforeAdvice  AfterReturningAdvice ThrowsAdvice
		// register well-known adapters
		registerAdvisorAdapter(new BeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}
    
    public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
        if (adviceObject instanceof Advisor) {
            return (Advisor) adviceObject;
        }

        if (!(adviceObject instanceof Advice)) {
            throw new UnknownAdviceTypeException(adviceObject);
        }
        Advice advice = (Advice) adviceObject;

        if (advice instanceof Interceptor) {
            // So well-known it doesn't even need an adapter
            return new DefaultPointcutAdvisor(advice);
        }

        // 遍历内置的 advice adapters
        for (int i = 0; i < this.adapters.size(); i++) {
            // Check that it is supported
            AdvisorAdapter adapter = (AdvisorAdapter) this.adapters.get(i);
            // 判断当前 adapter 是否支付当前 advice
            if (adapter.supportsAdvice(advice)) {
                // 如果支持的话，返回一个 DefaultPointcutAdvisor
                return new DefaultPointcutAdvisor(advice);
            }
        }
        throw new UnknownAdviceTypeException(advice);
    }
}
```



```java
public class DefaultPointcutAdvisor implements PointcutAdvisor, Ordered {

	private int order = Integer.MAX_VALUE;

	private Pointcut pointcut;
	
	private Advice advice;
	
	public DefaultPointcutAdvisor() {
	}
	
    // 默认匹配所有的类及其所有方法
	public DefaultPointcutAdvisor(Advice advice) {
		this(Pointcut.TRUE, advice);
	}
	
     // 指定匹配pointcut指向的类及其方法
	public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice) {
		this.pointcut = pointcut;
		this.advice = advice;
	}
}

 // 默认匹配所有的类及其所有方法,在PointcutAdvisor类中
Pointcut TRUE = new Pointcut() {

	public ClassFilter getClassFilter() {
		return ClassFilter.TRUE;
	}

	public MethodMatcher getMethodMatcher() {
		return MethodMatcher.TRUE;
	}

	public String toString() {
		return "Pointcut.TRUE";
	}
};
```

从 `DefaultPointcutAdvisor` 的实例可以看出创建 advisor的过程实际就是将 advice和 pointcut绑定的过程，且 默认的 pointcut 是拦截所有类下及其所有方法。也就是说createAdvisorChain就是将配置的advice和目标类及其方法进行绑定。下面再看`ProxyFactoryBean `的getSingletonInstance方法。调用getBean实质也是调用getSingletonInstance方法（单例）。

```java
private Object getSingletonInstance() {
    if (this.singletonInstance == null) {
        // This object can configure the proxy directly if it's
        // being used as a singleton.
        this.singletonInstance = createAopProxy().getProxy();
    }
    return this.singletonInstance;
}

protected synchronized AopProxy createAopProxy() {
    if (!isActive) {
        activate();
    }

    // getAopProxyFactory默认返回DefaultAopProxyFactory
    return getAopProxyFactory().createAopProxy(this);
}
```

`DefaultAopProxyFactory`的createAopProxy方法

```java
public AopProxy createAopProxy(AdvisedSupport advisedSupport) throws AopConfigException {
    boolean useCglib = advisedSupport.getOptimize() || advisedSupport.getProxyTargetClass() || advisedSupport.getProxiedInterfaces().length == 0;
    if (useCglib) {
        return CglibProxyFactory.createCglibProxy(advisedSupport);
    }
    else {
        // Depends on whether we have expose proxy or frozen or static ts
        return new JdkDynamicAopProxy(advisedSupport);
    }
}
```

前所例子中会使用`JdkDynamicAopProxy`，先看其实现

```java
protected JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config == null)
        throw new AopConfigException("Cannot create AopProxy with null ProxyConfig");
    if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE)
        throw new AopConfigException("Cannot create AopProxy with no advisors and no target source");
    this.advised = config;
}

public Object getProxy() {
    return getProxy(Thread.currentThread().getContextClassLoader());
}

/**
* Creates a new Proxy object for the given object, proxying
* the given interface. Uses the given class loader.
*/
public Object getProxy(ClassLoader cl) {
    Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
    // jdk动态代理的标准写法,this为实现了InvocationHandler接口的对象,意味着方法的调用会被转发到this的
    // invoke方法上
    return Proxy.newProxyInstance(cl, proxiedInterfaces, this);
}

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation = null;
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = advised.targetSource;
    Class targetClass = null;
    Object target = null;		

    try {
        // Try special rules for equals() method and implementation of the
        // Advised AOP configuration interface

        // Short-circuit expensive Method.equals() call, as Object.equals() isn't overloaded
        if (method.getDeclaringClass() == Object.class && "equals".equals(method.getName())) {
            // What if equals throws exception!?
            // This class implements the equals() method itself
            return new Boolean(equals(args[0]));
        }
        else if (Advised.class == method.getDeclaringClass()) {
            // Service invocations on ProxyConfig with the proxy config
            return AopProxyUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal = null;

        // TargetSource 单独分析，值得学习
        // May be null. Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }

        if (this.advised.exposeProxy) {
            // 暴露到当前线程
            // Make invocation available if necessary
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // 获取advice调用链
        // Get the interception chain for this method
        List chain = this.advised.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this.advised, proxy, method, targetClass);

        // Check whether we have any advice. If we don't, we can fallback on
        // direct reflective invocation of the target, and avoid creating a MethodInvocation
        if (chain.isEmpty()) {
            // 直接调用目标方法
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying
            retVal = AopProxyUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        else {
            // We need to create a method invocation...
            //invocation = advised.getMethodInvocationFactory().getMethodInvocation(proxy, method, targetClass, target, args, chain, advised);

            invocation = new ReflectiveMethodInvocation(proxy, target,
                                                        method, args, targetClass, chain);

            // Proceed to the joinpoint through the interceptor chain
            retVal = invocation.proceed();
        }

        // 如果方法返回的是其所在对象,返回其代理
        // 例子中 test1.returnsThis() 就会走到这个分支,避免target被错误暴露
        // Massage return value if necessary
        if (retVal != null && retVal == target) {
            // Special case: it returned "this"
            // Note that we can't help if the target sets
            // a reference to itself in another returned object
            retVal = proxy;
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            // Restore old proxy
            AopContext.setCurrentProxy(oldProxy);
        }

        //if (invocation != null) {
        //	advised.getMethodInvocationFactory().release(invocation);
        //}
    }
}
```

this.advised.advisorChainFactory默认获取的是`HashMapCachingAdvisorChainFactory`对象，观察其getInterceptorsAndDynamicInterceptionAdvice方法

```java
public List getInterceptorsAndDynamicInterceptionAdvice(Advised config, Object proxy, Method method, Class targetClass) {
    // methodCache 是IdentityHashMap,比较相等是引用相等
    // 随处可见的cache
    List cached = (List) this.methodCache.get(method);
    if (cached == null) {
        // Recalculate
        cached = AdvisorChainFactoryUtils.calculateInterceptorsAndDynamicInterceptionAdvice(config, proxy, method, targetClass);
        this.methodCache.put(method, cached);
    }
    return cached;
}
```

在看AdvisorChainFactoryUtils的calculateInterceptorsAndDynamicInterceptionAdvice方法

```java
public static List calculateInterceptorsAndDynamicInterceptionAdvice(Advised config, Object proxy, Method method, Class targetClass) {
    List interceptors = new ArrayList(config.getAdvisors().length);
    for (int i = 0; i < config.getAdvisors().length; i++) {
        Advisor advisor = config.getAdvisors()[i];
        if (advisor instanceof PointcutAdvisor) {
            // Add it conditionally
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            // 判断pointcut是否支持当前 target class 
            if (pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
                // 将advisor包装成interceptor
                MethodInterceptor interceptor = (MethodInterceptor) GlobalAdvisorAdapterRegistry.getInstance().getInterceptor(advisor);
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                // 是否匹配target class 的 method
                if (mm.matches(method, targetClass)) {
                    // 判断是否需要运行时才能判断方法的匹配
                    if (mm.isRuntime()) {
                        // Creating a new object instance in the getInterceptor() method
                        // isn't a problem as we normally cache created chains
                        interceptors.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm) );
                    }
                    else {
                        // 例子中会走到此分支
                        interceptors.add(interceptor);
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (ia.getClassFilter().matches(targetClass)) {
                MethodInterceptor interceptor = (MethodInterceptor) GlobalAdvisorAdapterRegistry.getInstance().getInterceptor(advisor);
                interceptors.add(interceptor);
            }
        }
    }	// for
    return interceptors;
}	// calculateInterceptorsAndDynamicInterceptionAdvice
```

此后chain不空，又被包装到`ReflectiveMethodInvocation`对象中，并调用其proceed方法

```java
public ReflectiveMethodInvocation(Object proxy, Object target, 
                                  Method m, Object[] arguments,
                                  Class targetClass, List interceptorsAndDynamicMethodMatchers) {
    this.proxy = proxy;
    this.target = target;
    this.targetClass = targetClass;
    this.method = m;
    this.arguments = arguments;
    this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
}

public Object proceed() throws Throwable {
    // 	当执行到最后一个拦截器的时候就会调用目标方法
    //	We start with an index of -1 and increment early
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        // 目标方法
        return invokeJoinpoint();
    }

    // 顺序调用Interceptor
    Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match
        InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        else {
            // Dynamic matching failed
            // Skip this interceptor and invoke the next in the chain
            return proceed();
        }
    }
    else {
        // 终于调用了interceptor！
        // It's an interceptor so we just invoke it: the pointcut will have
        // been evaluated statically before this object was constructed
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}

protected Object invokeJoinpoint() throws Throwable {
    return AopProxyUtils.invokeJoinpointUsingReflection(target, method, arguments);
}
```

```java
final class MethodBeforeAdviceInterceptor implements MethodInterceptor {
	
	private MethodBeforeAdvice advice;
	
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		this.advice = advice;
	}

	/**
	 * @see org.aopalliance.intercept.MethodInterceptor#invoke(org.aopalliance.intercept.MethodInvocation)
	 */
	public Object invoke(MethodInvocation mi) throws Throwable {
		advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}
	
}
```

AopProxyUtils

```java
public static Object invokeJoinpointUsingReflection(Object target, Method m, Object[] args) throws Throwable {
    //	Use reflection to invoke the method
    try {
        // 调用method
        Object rval = m.invoke(target, args);
        return rval;
    }
    catch (InvocationTargetException ex) {
        // Invoked method threw a checked exception. 
        // We must rethrow it. The client won't see the interceptor.
        throw ex.getTargetException();
    }
    catch (IllegalArgumentException ex) {
        throw new AspectException("AOP configuration seems to be invalid: tried calling " + m + " on [" + target + "]: " +  ex);
    }
    catch (IllegalAccessException ex) {
        throw new AspectException("Couldn't access method " + m, ex);
    }
}
```

由上述可见，Aop的执行过程就是：

- 获取当前目标方法的 interceptor chain

1. 遍历 advisor ，判断当前目标类和目标方法是否匹配 advisor 对应的 ponitcut
2. 通过匹配的 advisor 对应的 advice 匹配对应的 advisorAdapter , 进而获取对应的 methodInterceptor

- 执行拦截器

- 执行目标方法

---

Spring AOP 中的对象关系小结下：

- Advisor : 翻译是顾问，简单理解其就是一个 Aspect (切面); 其内部绑定了对应的 Pointcut(切入点) 和 Advice(通知)。
- Advisor Chain ： 切面链，是一系列的切面的集合。
- Advice : 通知，是对拦截方法的增强处理；在 1.0 版本中包含 BeforeAdivce, AfterReturningAdvice, ThrowsAdvice; 其面向的是用户。
- MethodInterceptor : 方法拦截器，是 Advice 的执行者; 与 Advice 是一一对应的。
- AdvisorAdapter : Advice 的适配器，是 Advice 和 MethodInterceptor 匹配的纽带。
- AdvisorAdapterRegistry : 是 AdvisorAdapter 的注册中心，内置了 BeforeAdviceAdapter, AfterReturnAdviceAdapter, ThrowsAdviceAdapter； 用来将 Advice wrap 成一个 Advisor 并提供获取 Advice 对应的 MethodInterceptor。

再从prototype类型的bean代理梳理一下处理逻辑。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    
    <bean id="prototypeTarget" class="com.youxiao.doc.aop.SideEffectBean" singleton="false">
        <property name="count"><value>10</value></property>
    </bean>

    <bean id="beforeAdvice" class="com.youxiao.doc.aop.BeforeAdvice">
    </bean>

    <bean id="prototype"
          class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="interceptorNames"><value>beforeAdvice,prototypeTarget</value></property>
        <property name="singleton"><value>false</value></property>
        <property name="proxyTargetClass"><value>true</value></property>
    </bean>
</beans>
```

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class PrototypeMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("prototype.xml");
        SideEffectBean prototype = (SideEffectBean) context.getBean("prototype");
        System.out.println(prototype.getCount());
    }
}

import org.springframework.aop.MethodBeforeAdvice;
import java.lang.reflect.Method;

public class BeforeAdvice implements MethodBeforeAdvice {

    @Override
    public void before(Method m, Object[] args, Object target) throws Throwable {
        System.out.println("do before advice ....");
    }
}

public class SideEffectBean {
	
	private int count;
	
	public void setCount(int count) {
		this.count = count;
	}
	
	public int getCount() {
		return this.count;
	}
	
	public void doWork() {
		++count;
	}
}
```

在构建prototype bean时并没有指定target，而是通过ProxyFactoryBean的addAdvisor方法设置

```java
private void addAdvisor(Object next, String name) {
    // 此时advisor实质是SingletonTargetSource对象
    Object advisor = namedBeanToAdvisorOrTargetSource(next);
    if (advisor instanceof Advisor) {
		// 省略
    } else {
        // 设定targetSource
        setTargetSource((TargetSource) advisor);
        // save target name
        this.targetName = name;
    }
}

// 
private Object namedBeanToAdvisorOrTargetSource(Object next) {
    try {
        // 不是Advisor也不是Advice
        Advisor adv = GlobalAdvisorAdapterRegistry.getInstance().wrap(next);
        return adv;
    }
    catch (UnknownAdviceTypeException ex) {
        // TODO consider checking that it's the last in the list?
        if (next instanceof TargetSource) {
            return (TargetSource) next;
        }
        else {
            // 通过此处返回
            // It's not a pointcut or interceptor.
            // It's a bean that needs an invoker around it.
            return new SingletonTargetSource(next);
        }
    }
}
```

由于并没有设置代理接口，且非单例，和前述例子不同。以下从调用开始观察`FactoryBean`对其不同的处理。

```java
public Object getObject() throws BeansException {
    return (this.singleton) ? getSingletonInstance() : newPrototypeInstance();
}

private Object newPrototypeInstance() {
    refreshAdvisorChain();
    refreshTarget();
    // In the case of a prototype, we need to give the proxy
    // an independent instance of the configuration.
    AdvisedSupport copy = new AdvisedSupport();
    // 深度复制现有配置
    copy.copyConfigurationFrom(this);
    // createAopProxy().getProxy()和前述单例处理逻辑统一
    return copy.createAopProxy().getProxy();
}

private void refreshAdvisorChain() {
    Advisor[] advisors = getAdvisors();
    for (int i = 0; i < advisors.length; i++) {
        String beanName = (String) this.sourceMap.get(advisors[i]);
        if (beanName != null) {
            // 获取 advise bean
            Object bean = this.beanFactory.getBean(beanName);
            // 返回新的DefaultPointcutAdvisor
            Object refreshedAdvisor = namedBeanToAdvisorOrTargetSource(bean);
            // might have just refreshed target source
            if (refreshedAdvisor instanceof Advisor) {
                // 替换advisor
                // What about aspect interfaces!? we're only updating
                replaceAdvisor(advisors[i], (Advisor) refreshedAdvisor);
            }
            else {
                setTargetSource((TargetSource) refreshedAdvisor);
            }
            // 替换beanName对应的advisor
            // keep name mapping up to date
            this.sourceMap.put(refreshedAdvisor, beanName);
        }
        else {
            // We can't throw an exception here, as the user may have added additional
            // pointcuts programmatically we don't know about
            logger.info("Cannot find bean name for Advisor [" + advisors[i] + 
                        "] when refreshing advisor chain");
        }
    }
}

private void refreshTarget() {
    if (this.targetName == null) {
        throw new AopConfigException("Target name cannot be null when refreshing!");
    }
    // 刷新target,如果target是prototype那么target会重新生成
    Object target = this.beanFactory.getBean(this.targetName);
    setTarget(target);
}
```

此时getProxy获取的不再是jdk动态代理`JdkDynamicAopProxy`对象而是`Cglib2AopProxy`对象。

```java
protected Cglib2AopProxy(AdvisedSupport config) throws AopConfigException {
    if (config == null)
        throw new AopConfigException("Cannot create AopProxy with null ProxyConfig");
    if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE)
        throw new AopConfigException("Cannot create AopProxy with no advisors and no target source");
    this.advised = config;
    if (this.advised.getTargetSource().getTargetClass() == null) {
        throw new AopConfigException("Either an interface or a target is required for proxy creation");
    }
}

// 对target方法的调用会被转发到此方法上
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    MethodInvocation invocation = null;
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = advised.targetSource;
    Class targetClass = null;//targetSource.getTargetClass();
    Object target = null;		

    try {
        // Try special rules for equals() method and implementation of the
        // ProxyConfig AOP configuration interface
        if (isEqualsMethod(method)) {
            // This class implements the equals() method itself
            // We don't need to use reflection
            return new Boolean(equals(args[0]));
        }
        else if (Advised.class == method.getDeclaringClass()) {
            // Service invocations on ProxyConfig with the proxy config
            return AopProxyUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal = null;

        // May be null. Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }

        if (this.advised.exposeProxy) {
            // Make invocation available if necessary
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        List chain = advised.getAdvisorChainFactory().getInterceptorsAndDynamicInterceptionAdvice(this.advised, proxy, method, targetClass);

        // Check whether we only have one InvokerInterceptor: that is, no real advice,
        // but just reflective invocation of the target.
        if (chain.isEmpty()) {
            // 直接调用target方法
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying
            retVal = methodProxy.invoke(target, args);
        }
        else {
            // 参考前述ReflectiveMethodInvocation分析
            // We need to create a method invocation...
            invocation = new MethodInvocationImpl(proxy, target, method, args, 
                                                  targetClass, chain, methodProxy);

            // If we get here, we need to create a MethodInvocation
            retVal = invocation.proceed();
        }

        retVal = massageReturnTypeIfNecessary(proxy, target, retVal);
        return retVal;
    }
    catch (Throwable t) {
        // In CGLIB2, unlike CGLIB 1, it's necessary to wrap
        // undeclared throwable exceptions. As we don't care about JDK 1.2
        // compatibility, we use java.lang.reflect.UndeclaredThrowableException.
        if ( (t instanceof Exception) && !(t instanceof RuntimeException)) {
            // It's a checked exception: we must check it's legal
            Class[] permittedThrows = method.getExceptionTypes();
            for (int i = 0; i < permittedThrows.length; i++) {
                if (permittedThrows[i].isAssignableFrom(t.getClass())) {
                    throw t;
                }
            }
            throw new UndeclaredThrowableException(t);
        }

        // It's not a checked exception, so we can rethrow it
        throw t;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource
            targetSource.releaseTarget(target);
        }

        if (setProxyContext) {
            // Restore old proxy
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}	// intercept
```

```java
// ReflectiveMethodInvocation在JdkDynamicAopProxy中也用运用
private static class MethodInvocationImpl extends ReflectiveMethodInvocation {

    private MethodProxy methodProxy;

    public MethodInvocationImpl(Object proxy, Object target, Method m, Object[] arguments, Class targetClass, List interceptorsAndDynamicMethodMatchers,
MethodProxy methodProxy) {
        super(proxy, target, m, arguments, targetClass, interceptorsAndDynamicMethodMatchers);
        this.methodProxy = methodProxy;
    }

    // 重载ReflectiveMethodInvocation的invokeJoinpoint方法,该为cglib的调用方式
    protected Object invokeJoinpoint() throws Throwable {
        return methodProxy.invoke(target, arguments);
    }
}
```

关于`TargetSources`，是spring在target的一种封装，除了简单的`SingletonTargetSource`，还提供了`HotSwappableTargetSource`、`PrototypeTargetSource`、`CommonsPoolTargetSource`、`ThreadLocalTargetSource`等多种实现，值得借鉴。以`PrototypeTargetSource`为例，摘自spring test代码

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>

	<!-- Simple target -->
	<bean id="test" class="org.springframework.aop.interceptor.SideEffectBean">	
		<property name="count"><value>10</value></property>
	</bean>
	
	<bean id="prototypeTest" class="org.springframework.aop.interceptor.SideEffectBean" 
		singleton="false">	
		<property name="count"><value>10</value></property>
	</bean>
	
	<bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">	
		<property name="targetBeanName"><value>prototypeTest</value></property>
	</bean>
	
	<bean id="debugInterceptor" class="org.springframework.aop.interceptor.NopInterceptor" />
	
	<bean id="singleton" class="org.springframework.aop.framework.ProxyFactoryBean">	
		<property name="interceptorNames"><value>debugInterceptor,test</value></property>
	</bean>
	
	<!--
		This will create a bean that creates a new
		target on each invocation.
	-->
	<bean id="prototype" class="org.springframework.aop.framework.ProxyFactoryBean">	
		<property name="targetSource"><ref local="prototypeTargetSource"/></property>
		<property name="interceptorNames"><value>debugInterceptor</value></property>
	</bean>
</beans>	
```

```java
public void testPrototypeAndSingletonBehaveDifferently() {
    SideEffectBean singleton = (SideEffectBean) beanFactory.getBean("singleton");
    assertEquals(INITIAL_COUNT, singleton.getCount() );
    singleton.doWork();
    assertEquals(INITIAL_COUNT + 1, singleton.getCount() );

    SideEffectBean prototype = (SideEffectBean) beanFactory.getBean("prototype");
    assertEquals(INITIAL_COUNT, prototype.getCount() );
    prototype.doWork();
    // 每次获取的count都是新建SideEffectBean的值
    assertEquals(INITIAL_COUNT, prototype.getCount() );
}
```

观察一下其实现`PrototypeTargetSource`

```
public final class PrototypeTargetSource extends AbstractPrototypeTargetSource {

   public Object getTarget() {
      return newPrototypeInstance();
   }
   
   public void releaseTarget(Object target) {
      // Do nothing
   }
}
```

`AbstractPrototypeTargetSource`

```java
protected Object newPrototypeInstance() {
    return this.owningBeanFactory.getBean(this.targetBeanName);
}
```

由于targetBeanName是非单例，意味着每次getBean的时候都会创建targetBeanName的实例。
