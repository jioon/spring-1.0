spring-transaction

介绍transaction之前，先回顾一下事务几个概念。

隔离级别

TransactionDefinition 接口中定义了五个表示隔离级别的常量：

- ISOLATION_DEFAULT：使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别。
- ISOLATION_READ_UNCOMMITTED：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。
- ISOLATION_READ_COMMITTED：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。
- ISOLATION_REPEATABLE_READ：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
- ISOLATION_SERIALIZABLE：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

spring 事务传播行为，解决业务层方法之间互相调用的事务问题

- PROPAGATION_REQUIRED：支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。

- PROPAGATION_SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行。

- PROPAGATION_MANDATORY：支持当前事务，如果当前没有事务，就抛出异常。 

- PROPAGATION_REQUIRES_NEW：新建事务，如果当前存在事务，把当前事务挂起。 

- PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 

- PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。

- PROPAGATION_NESTED： 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。

  前面的六种事务传播行为是 Spring 从 EJB 中引入的，他们共享相同的概念。而 **PROPAGATION_NESTED** 是 Spring 所特有的，且1.0中并没有这个行为。
  
  参考[详解spring事务属性](https://www.iteye.com/topic/78674)

从简单的使用开始。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>

    <!-- Simple target -->
    <bean id="target" class="com.youxiao.doc.transaction.TestBean">
        <property name="name">
            <value>custom</value>
        </property>
    </bean>

    <bean id="mockMan" class="com.youxiao.doc.transaction.PlatformTransactionManagerFacade"/>

    <bean id="txInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor">
        <property name="transactionManager">
            <ref local="mockMan"/>
        </property>
        <property name="transactionAttributeSource">
            <value>
                com.youxiao.doc.transaction.ITestBean.set*=PROPAGATION_REQUIRED
            </value>
        </property>
    </bean>

    <bean id="proxyFactory1" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces">
            <value>com.youxiao.doc.transaction.ITestBean</value>
        </property>
        <property name="interceptorNames">
            <list>
                <value>txInterceptor</value>
                <value>target</value>
            </list>
        </property>
    </bean>
</beans>

```

```java
public interface ITestBean {

	int getAge();

	void setAge(int age);

	String getName();

	void setName(String name);

	Object returnsThis();
}

public class TestBean implements ITestBean {
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
        System.out.println("setAge");
    }

    @Override
    public String getName() {
        System.out.println("getName");
        return name;
    }

    @Override
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public Object returnsThis() {
        return this;
    }
}

import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionException;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionStatus;

public class PlatformTransactionManagerFacade implements PlatformTransactionManager {
    private TransactionStatus ts = new DefaultTransactionStatus(null, true, false, false, false, null);

    @Override
    public TransactionStatus getTransaction(TransactionDefinition definition)
            throws TransactionException {
        System.out.println("getTransaction");
        return ts;
    }

    @Override
    public void commit(TransactionStatus status) throws TransactionException {
        System.out.println("commit");
    }

    @Override
    public void rollback(TransactionStatus status) throws TransactionException {
        System.out.println("rollback");
    }
}

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class TransactionMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("transactionalBeanFactory.xml");
        ITestBean bean = (ITestBean) context.getBean("proxyFactory1", ITestBean.class);
        System.out.println(bean.getName());
        bean.setAge(123);
    }
}
```

输出为：

getName
custom
getTransaction
setAge
commit

从spring-aop分析过程来看，我们可以从`TransactionInterceptor`入手分析transaction流程。`TransactionInterceptor`需要2个成员，一个是`PlatformTransactionManager`，用以代理实际的事务操作，放在jdbc中分析。一个是`TransactionAttributeSource`，用以承载事务相关配置。

transactionAttributeSource是通过`TransactionAttributeSourceEditor`将xml中配置的String转换成`TransactionAttributeSource`设入`TransactionInterceptor`中。

```java
public class TransactionAttributeSourceEditor extends PropertyEditorSupport {
	public void setAsText(String s) throws IllegalArgumentException {
		MethodMapTransactionAttributeSource source = new MethodMapTransactionAttributeSource();
		if (s == null || "".equals(s)) {
			// Leave value in property editor null
		}
		else {
            // transactionAttributeSource 配置会被装载成properties
			// Use properties editor to tokenize the hold string
			PropertiesEditor propertiesEditor = new PropertiesEditor();
			propertiesEditor.setAsText(s);
			Properties props = (Properties) propertiesEditor.getValue();

			// Now we have properties, process each one individually
			TransactionAttributeEditor tae = new TransactionAttributeEditor();
			for (Iterator iter = props.keySet().iterator(); iter.hasNext();) {
				String name = (String) iter.next();
				String value = props.getProperty(name);

				// Convert value to a transaction attribute
				tae.setAsText(value);
				TransactionAttribute attr = (TransactionAttribute) tae.getValue();

				// Register name and attribute
				source.addTransactionalMethod(name, attr);
			}
		}
		setValue(source);
	}
}
```

对于`MethodMapTransactionAttributeSource`就是用记录方法上的事务配置类。

```java
public void addTransactionalMethod(String name, TransactionAttribute attr) {
    int lastDotIndex = name.lastIndexOf(".");
    if (lastDotIndex == -1) {
        throw new TransactionUsageException("'" + name + "' is not a valid method name: format is FQN.methodName");
    }
    String className = name.substring(0, lastDotIndex);
    String methodName = name.substring(lastDotIndex + 1);
    try {
        Class clazz = Class.forName(className, true, Thread.currentThread().getContextClassLoader());
        addTransactionalMethod(clazz, methodName, attr);
    }
    catch (ClassNotFoundException ex) {
        throw new TransactionUsageException("Class '" + className + "' not found");
    }
}

public void addTransactionalMethod(Class clazz, String mappedName, TransactionAttribute attr) {
    String name = clazz.getName() + '.'  + mappedName;

    // TODO address method overloading? At present this will
    // simply match all methods that have the given name.
    // Consider EJB syntax (int, String) etc.?
    Method[] methods = clazz.getDeclaredMethods();
    List matchingMethods = new ArrayList();
    for (int i = 0; i < methods.length; i++) {
        // 可能有多个方法匹配 因为支持通配符*
        if (methods[i].getName().equals(mappedName) || isMatch(methods[i].getName(), mappedName)) {
            matchingMethods.add(methods[i]);
        }
    }
    if (matchingMethods.isEmpty()) {
        throw new TransactionUsageException("Couldn't find method '" + mappedName +
"' on class [" + clazz.getName() + "]");
    }

    // register all matching methods
    for (Iterator it = matchingMethods.iterator(); it.hasNext();) {
        Method method = (Method) it.next();
        String regMethodName = (String) this.nameMap.get(method);
        // 同一个方法更具体的配置才有效
        if (regMethodName == null || (!regMethodName.equals(name) && regMethodName.length() <= name.length())) {
            // no already registered method name, or more specific
            // method name specification now -> (re-)register method
            this.nameMap.put(method, name);
            // 加入到map中
            addTransactionalMethod(method, attr);
        }
        else {
            // log
        }
    }
}

// 只支持前后通配符
protected boolean isMatch(String methodName, String mappedName) {
    return (mappedName.endsWith("*") && methodName.startsWith(mappedName.substring(0, mappedName.length() - 1))) ||
        (mappedName.startsWith("*") && methodName.endsWith(mappedName.substring(1, mappedName.length())));
}
```

`TransactionAttributeEditor`

```java
public class TransactionAttributeEditor extends PropertyEditorSupport {

	/**
	 * Format is PROPAGATION_NAME,ISOLATION_NAME,readOnly,+Exception1,-Exception2.
	 * Null or the empty string means that the method is non transactional.
	 * @see java.beans.PropertyEditor#setAsText(java.lang.String)
	 */
	public void setAsText(String s) throws IllegalArgumentException {
		if (s == null || "".equals(s)) {
			setValue(null);
		}
		else {	
			// tokenize it with ","
			String[] tokens = StringUtils.commaDelimitedListToStringArray(s);
			RuleBasedTransactionAttribute attr = new RuleBasedTransactionAttribute();

			for (int i = 0; i < tokens.length; i++) {
				String token = tokens[i];
                // 以 PROPAGATION 开头，则配置事务传播性
                // 默认PROPAGATION_REQUIRED
				if (token.startsWith(TransactionDefinition.PROPAGATION_CONSTANT_PREFIX)) {
					attr.setPropagationBehaviorName(tokens[i]);
				}
                // 以 ISOLATION 开头，则配置事务隔离级别
                // 默认ISOLATION_DEFAULT
				else if  (token.startsWith(TransactionDefinition.ISOLATION_CONSTANT_PREFIX)) {
					attr.setIsolationLevelName(tokens[i]);
				}
                // 以 timeout_ 开头，则设置事务超时时间
                // 默认-1
				else if (token.startsWith(DefaultTransactionAttribute.TIMEOUT_PREFIX)) {
					String value = token.substring(DefaultTransactionAttribute.TIMEOUT_PREFIX.length());
					attr.setTimeout(Integer.parseInt(value));
				}
                // 若等于 readOnly 则配置事务只读
                // 默认false
				else if (token.equals(DefaultTransactionAttribute.READ_ONLY_MARKER)) {
					attr.setReadOnly(true);
				}
                // 以 + 开头，则配置哪些异常下不回滚
				else if (token.startsWith(DefaultTransactionAttribute.COMMIT_RULE_PREFIX)) {
					attr.getRollbackRules().add(new NoRollbackRuleAttribute(token.substring(1)));
				}
                // 以 - 开头，则配置哪些异常下回滚
				else if (token.startsWith(DefaultTransactionAttribute.ROLLBACK_RULE_PREFIX)) {
					attr.getRollbackRules().add(new RollbackRuleAttribute(token.substring(1)));
				}
				else {
					throw new IllegalArgumentException("Illegal transaction token: " + token);
				}
			}

			setValue(attr);
		}
	}

}
```

也就是说完整的配置可以是：

```xml
<property name="transactionAttributeSource">
    <value>      com.youxiao.doc.transaction.ITestBean.set*=PROPAGATION_REQUIRED,ISOLATION_DEFAULT,timeout_10,readOnly,+Exception1,-Exception2
    </value>
</property>
```

按照之前spring-aop的分析，当代理对象在执行的时候会先获取当前方法所匹配的advisor，在需要拦截的方法调用时会触发`TransactionInterceptor`的invoke方法调用（默认会被包装成`DefaultPointcutAdvisor`，拦截所有类及方法）。

```java
public final Object invoke(MethodInvocation invocation) throws Throwable {
    // Work out the target class: may be null.
    // The TransactionAttributeSource should be passed the target class
    // as well as the method, which may be from an interface
    Class targetClass = (invocation.getThis() != null) ? invocation.getThis().getClass() : null;

    // if the transaction attribute is null, the method is non-transactional
    TransactionAttribute transAtt = this.transactionAttributeSource.getTransactionAttribute(invocation.getMethod(), targetClass);
    TransactionStatus status = null;
    TransactionStatus oldTransactionStatus = null;

    // create transaction if necessary
    if (transAtt != null) {
        // we need a transaction for this method
        // the transaction manager will flag an error if an incompatible tx already exists
        status = this.transactionManager.getTransaction(transAtt);

        // make the TransactionStatus available to callees
        oldTransactionStatus = (TransactionStatus) currentTransactionStatus.get();
        currentTransactionStatus.set(status);
    }
    else {
        // 如果transAtt为null 代表不是事务方法 不做特殊处理
        // it isn't a transactional method
    }

    // Invoke the next interceptor in the chain.
    // This will normally result in a target object being invoked.
    Object retVal = null;
    try {
        // 链式调用
        retVal = invocation.proceed();
    }
    catch (Throwable ex) {
        // target invocation exception
        if (status != null) {
            // 按需回滚
            onThrowable(invocation, transAtt, status, ex);
        }
        throw ex;
    }
    finally {
        if (transAtt != null) {
            // use stack to restore old transaction status if one was set
            currentTransactionStatus.set(oldTransactionStatus);
        }
    }
    if (status != null) {
        this.transactionManager.commit(status);
    }
    return retVal;
}
```

以上是直接使用`ProxyFactoryBean`实现事务操作，实际上有专门的factoryBean来处理，即`TransactionProxyFactoryBean`。

![TransactionProxyFactoryBean](https://github.com/jioon/spring-1.0/blob/master/transaction/image/TransactionProxyFactoryBean.jpg?raw=true)

`transactionalBeanFactory.xml`中增加配置

```xml
<bean id="proxyFactory2" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager">
        <ref local="mockMan"/>
    </property>
    <property name="target">
        <ref local="target"/>
    </property>
    <property name="transactionAttributes">
        <props>
            <prop key="set*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

调整main方法为

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class TransactionMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("transactionalBeanFactory.xml");
        ITestBean bean2 = (ITestBean) context.getBean("proxyFactory2", ITestBean.class);
        System.out.println(bean2.getName());
        bean2.setAge(123);
    }
}
```

其输出为：

getName
custom
getTransaction
setAge
commit

和前述一致。

先看proxyFactory2属性填充，transactionManager target只是简单的赋值，先看transactionAttributes

```java
public void setTransactionAttributes(Properties transactionAttributes) {
    NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
    tas.setProperties(transactionAttributes);
    this.transactionAttributeSource = tas;
}
```

此时TransactionAttributeSource已替换成`NameMatchTransactionAttributeSource`，而不是`MethodMapTransactionAttributeSource`。

```java
public void setProperties(Properties transactionAttributes) {
    TransactionAttributeEditor tae = new TransactionAttributeEditor();
    for (Iterator it = transactionAttributes.keySet().iterator(); it.hasNext(); ) {
        String methodName = (String) it.next();
        String value = transactionAttributes.getProperty(methodName);
        // 解析标签设置 隔离级别 传播行为 超时 只读 回滚,和前述一致
        tae.setAsText(value);
        TransactionAttribute attr = (TransactionAttribute) tae.getValue();
        // 记录到map中
        addTransactionalMethod(methodName, attr);
    }
}
```

由于proxyFactory2实现了`InitializingBean`接口，那么一定会调用afterPropertiesSet方法

```java
public void afterPropertiesSet() throws AopConfigException {
    if (this.target == null) {
        throw new AopConfigException("Target must be set");
    }

    if (this.transactionAttributeSource == null) {
        throw new AopConfigException("Either 'transactionAttributeSource' or 'transactionAttributes' is required: " + "If there are no transactional methods, don't use a transactional proxy.");
    }

    //和前述直接配置transactionInterceptor一致
    TransactionInterceptor transactionInterceptor = new TransactionInterceptor();
    transactionInterceptor.setTransactionManager(this.transactionManager);
    transactionInterceptor.setTransactionAttributeSource(this.transactionAttributeSource);
    // 只是校验 transactionManager transactionAttributeSource是否存在 
    transactionInterceptor.afterPropertiesSet();

    ProxyFactory proxyFactory = new ProxyFactory();
    // 是否配置了前置拦截
    if (this.preInterceptors != null) {
        for (int i = 0; i < this.preInterceptors.length; i++) {
            proxyFactory.addAdvisor(GlobalAdvisorAdapterRegistry.getInstance().wrap(this.preInterceptors[i]));
        }
    }

    if (this.pointcut != null) {
        // 如果配置了 pointcut 切入点，则按配置的 pointcut 创建 advisor
        Advisor advice = new DefaultPointcutAdvisor(this.pointcut, transactionInterceptor);
        proxyFactory.addAdvisor(advice);
    }
    else {
        // 组装默认的advisor
        // 匹配所有的类,方法匹配被重载
        // rely on default pointcut
        proxyFactory.addAdvisor(new TransactionAttributeSourceAdvisor(transactionInterceptor));
        // could just do the following, but it's usually less efficient because of AOP advice chain caching
        // proxyFactory.addInterceptor(transactionInterceptor);
    }

    // 是否配置了后置拦截
    if (this.postInterceptors != null) {
        for (int i = 0; i < this.postInterceptors.length; i++) {
            proxyFactory.addAdvisor(GlobalAdvisorAdapterRegistry.getInstance().wrap(this.postInterceptors[i]));
        }
    }

    proxyFactory.copyFrom(this);
    proxyFactory.setTargetSource(createTargetSource(this.target));
    if (this.proxyInterfaces != null) {
        proxyFactory.setInterfaces(this.proxyInterfaces);
    }
    else if (!getProxyTargetClass()) {
        // rely on AOP infrastructure to tell us what interfaces to proxy
        proxyFactory.setInterfaces(AopUtils.getAllInterfaces(this.target));
    }
    this.proxy = proxyFactory.getProxy();
}
```

`TransactionAttributeSourceAdvisor`

```java
public class TransactionAttributeSourceAdvisor extends StaticMethodMatcherPointcutAdvisor {
	
	private TransactionAttributeSource transactionAttributeSource;
	
	public TransactionAttributeSourceAdvisor(TransactionInterceptor ti) {
		super(ti);
		if (ti.getTransactionAttributeSource() == null) {
			throw new AopConfigException("Cannot construct a TransactionAttributeSourceAdvisor using a " + "TransactionInterceptor that has no TransactionAttributeSource configured");
		}
		this.transactionAttributeSource = ti.getTransactionAttributeSource();
	}

    // MethodMatcher matches
    // 交给transactionAttributeSource来匹配,就是NameMatchTransactionAttributeSource
	public boolean matches(Method m, Class targetClass) {
		return (this.transactionAttributeSource.getTransactionAttribute(m, targetClass) != null);
	}
}
```

`NameMatchTransactionAttributeSource`

```java
public TransactionAttribute getTransactionAttribute(Method method, Class targetClass) {
    String methodName = method.getName();
    TransactionAttribute attr = (TransactionAttribute) this.nameMap.get(methodName);
    if (attr != null) {
        return attr;
    }
    else {
        // look up most specific name match
        String bestNameMatch = null;
        for (Iterator it = this.nameMap.keySet().iterator(); it.hasNext();) {
            String mappedName = (String) it.next();
            // 寻找最具体的方法
            if (isMatch(methodName, mappedName) &&
                (bestNameMatch == null || bestNameMatch.length() <= mappedName.length())) {
                attr = (TransactionAttribute) this.nameMap.get(mappedName);
                bestNameMatch = mappedName;
            }
        }
        return attr;
    }
}

// 支持前后通配符
protected boolean isMatch(String methodName, String mappedName) {
    return (mappedName.endsWith("*") && methodName.startsWith(mappedName.substring(0, mappedName.length() - 1))) ||
        (mappedName.startsWith("*") && methodName.endsWith(mappedName.substring(1, mappedName.length())));
}
```

拿到proxy后，后续方法的调用就和前述分析一致了。

再来看一个例子。

修改`transactionalBeanFactory.xml` 为：

```xml
 <!-- Simple target -->
<bean id="target" class="com.youxiao.doc.transaction.TestBean">
    <property name="name">
        <value>custom</value>
    </property>
    <property name="age">
        <value>666</value>
    </property>
</bean>

<bean id="mockMan" class="com.youxiao.doc.transaction.PlatformTransactionManagerFacade"/>

<bean id="proxyFactory3" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager">
        <ref local="mockMan"/>
    </property>
    <property name="target">
        <ref local="target"/>
    </property>
    <property name="proxyTargetClass">
        <value>true</value>
    </property>
    <property name="transactionAttributes">
        <props>
             <prop key="set*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
    <property name="pointcut">
        <ref local="txnInvocationPointcut"/>
    </property>
    <property name="preInterceptors">
        <list>
            <ref local="preInvocationInterceptor"/>
        </list>
    </property>
    <property name="postInterceptors">
        <list>
            <ref local="postInvocationInterceptor"/>
        </list>
    </property>
</bean>

<bean id="txnInvocationPointcut"
      class="com.youxiao.doc.transaction.InvocationPointcut"
      />

<bean id="preInvocationInterceptor"
      class="com.youxiao.doc.transaction.InvocationPreInterceptor"/>

<bean id="postInvocationInterceptor"
      class="com.youxiao.doc.transaction.InvocationPostInterceptor"/>
```

```java
import org.springframework.aop.support.StaticMethodMatcherPointcut;
import java.lang.reflect.Method;
public class InvocationPointcut extends StaticMethodMatcherPointcut {

    @Override
    public boolean matches(Method method, Class clazz) {
        System.out.println("InvocationPointcut class : " + clazz.getName() + ",method : " + method.getName());
        return true;
    }
}

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
public class InvocationPreInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        System.out.println("InvocationPreInterceptor method name : " + methodInvocation.getMethod().getName());
        return methodInvocation.proceed();
    }
}

import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
public class InvocationPostInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        System.out.println("InvocationPostInterceptor method name : " + methodInvocation.getMethod().getName());
        return methodInvocation.proceed();
    }
}

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class TransactionMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("transactionalBeanFactory.xml");
        ITestBean bean3 = (ITestBean) context.getBean("proxyFactory3", ITestBean.class);
        System.out.println(bean3.getName());
        bean3.setAge(123);
    }
}
```

此时增加了自定义的pointcut，以及前置后置拦截。

其输出为：

InvocationPointcut class : com.youxiao.doc.transaction.TestBean,method : getName
InvocationPreInterceptor method name : getName
InvocationPostInterceptor method name : getName
getName
custom
InvocationPointcut class : com.youxiao.doc.transaction.TestBean,method : setAge
InvocationPreInterceptor method name : setAge
getTransaction
InvocationPostInterceptor method name : setAge
setAge
commit

我们再以真实的数据库操作为例，看看事务的操作。

修改`transactionalBeanFactory.xml` 为：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>
    <!-- 配置数据源 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
        <property name="driverClassName">
            <value>com.mysql.jdbc.Driver</value>
        </property>
        <property name="url">
            <value>url</value>
        </property>
        <property name="username">
            <value>username</value>
        </property>
        <property name="password">
            <value>password</value>
        </property>
    </bean>

    <!-- 配置事务管理 -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource">
            <ref bean="dataSource"></ref>
        </property>
    </bean>

    <!-- 配置 jdbcTemplate -->
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource">
            <ref bean="dataSource"/>
        </property>
    </bean>

    <bean id="userServiceTarget" class="com.youxiao.doc.transaction.UserService">
        <property name="jdbcTemplate">
            <ref bean="jdbcTemplate"></ref>
        </property>
    </bean>

    <bean id="proxyFactory4" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
        <property name="transactionManager">
            <ref local="transactionManager"/>
        </property>
        <property name="target">
            <ref local="userServiceTarget"/>
        </property>
        <property name="transactionAttributes">
            <props>
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>
</beans>
```

先建一张表

```sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

再修改代码为

```java
import org.springframework.jdbc.core.JdbcTemplate;
import java.util.List;

public class UserService {

    private JdbcTemplate jdbcTemplate;

    public List getList() {
        return jdbcTemplate.queryForList("select id,name from test");
    }

    public JdbcTemplate getJdbcTemplate() {
        return jdbcTemplate;
    }

    public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.jdbc.core.JdbcTemplate;
import java.util.List;
public class TransactionMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("transactionalBeanFactory.xml");
        UserService bean4 = (UserService) context.getBean("proxyFactory4", UserService.class);
        List list = bean4.getList();
        System.out.println(list.size());
    }
}
```

我们知道对方法的调用会被转发到`TransactionInterceptor`的invoke方法上

```java
public final Object invoke(MethodInvocation invocation) throws Throwable {
	...
    if (transAtt != null) {
        // 开启事务
        status = this.transactionManager.getTransaction(transAtt);
        ...
    }
    
  	...
	try {
		retVal = invocation.proceed();
	}
    catch (Throwable ex) {
        // target invocation exception
        if (status != null) {
            onThrowable(invocation, transAtt, status, ex);
        }
        throw ex;
    }
	...
    if (status != null) {
        // 提交事务
        this.transactionManager.commit(status);
    }
}

private void onThrowable(MethodInvocation invocation, TransactionAttribute txAtt,
                         TransactionStatus status, Throwable ex) {
    if (txAtt.rollbackOn(ex)) {
        try {
            // 回滚事务
            this.transactionManager.rollback(status);
        }
        catch (TransactionException tex) {
            throw tex;
        }
    }
    else {
        // Will still roll back if rollbackOnly is true
        this.transactionManager.commit(status);
    }
}
```

从`DataSourceTransactionManager`入手分析，其继承了抽象类`AbstractPlatformTransactionManager`，提供了公共方法getTransaction，标准的模板模式。

| 名词                              | 概念                                                         |
| --------------------------------- | :----------------------------------------------------------- |
| PlatformTransactionManager        | 事务管理器，管理事务的各生命周期方法                         |
| TransactionAttribute              | 事务属性, 包含隔离级别，传播行为,是否只读等信息              |
| TransactionStatus                 | 事务状态                                                     |
| TransactionSynchronization        | 事务同步回调，内含多个钩子方法                               |
| TransactionSynchronizationManager | 事务同步管理器，维护当前线程事务资源，事务同步回调，通过2个ThreadLocal实现，resources、synchronizations |

```java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    // 获取transaction对象
    Object transaction = doGetTransaction();
    if (definition == null) {
        // 采用默认的事务定义
        // use defaults
        definition = new DefaultTransactionDefinition();
    }

    // 判断当前上下文是否开启过事务
    if (isExistingTransaction(transaction)) {
        // 当前上下文开启过事务
		// 如果当前方法匹配的事务传播性为 PROPAGATION_NEVER 说明不需要事务则抛出异常
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
            throw new IllegalTransactionStateException("Transaction propagation 'never' but existing transaction found");
        }
        // 如果当前方法匹配的事务传播性为 PROPAGATION_NOT_SUPPORTED 说明该方法不应该运行在事务中，则将当前事务挂起
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
            Object suspendedResources = suspend(transaction);
            // transactionSynchronization默认就是SYNCHRONIZATION_ALWAYS，代表在任何情况下都需要同步
            boolean newSynchronization = (this.transactionSynchronization == SYNCHRONIZATION_ALWAYS);
            // 返回的事务状态为 不需要事务
            return newTransactionStatus(null, false, newSynchronization, definition.isReadOnly(), debugEnabled, suspendedResources);
        }
        // 如果当前方法匹配的事务传播性为 PROPAGATION_REQUIRES_NEW 表示当前方法必须运行在它自己的事务中，将已存在的事务挂起，重新开启事务
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
            // 挂起当前事务
            Object suspendedResources = suspend(transaction);
            // 重新开启新事务
            doBegin(transaction, definition);
            // 返回的事务状态为 新建事务
            boolean newSynchronization = (this.transactionSynchronization != SYNCHRONIZATION_NEVER);
            return newTransactionStatus(transaction, true, newSynchronization,definition.isReadOnly(), debugEnabled, suspendedResources);
        }
        else {
            boolean newSynchronization = (this.transactionSynchronization != SYNCHRONIZATION_NEVER);
            // 其他的传播行为 表示在已存在的事务中执行
            return newTransactionStatus(transaction, false, newSynchronization, definition.isReadOnly(), debugEnabled, null);
        }
    }

    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }
    
    // 如果传播性为 PROPAGATION_MANDATORY 说明必须在事务中执行，若当前没有事务的话则抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException("Transaction propagation 'mandatory' but no existing transaction found");
    }

    // 当前上下文不存在事务
	// 若传播性为 PROPAGATION_REQUIRED 或 PROPAGATION_REQUIRES_NEW 则开启新的事务执行
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
        definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        // 开启新的 connection 并取消自动提交，将 connection 绑定当前线程
        doBegin(transaction, definition);
        boolean newSynchronization = (this.transactionSynchronization != SYNCHRONIZATION_NEVER);
        return newTransactionStatus(transaction, true, newSynchronization,
                                    definition.isReadOnly(), debugEnabled, null);
    }
    else {
        // "empty" (-> no) transaction
        boolean newSynchronization = (this.transactionSynchronization == SYNCHRONIZATION_ALWAYS);
        // 返回事务状态为 不需要事务
        return newTransactionStatus(null, false, newSynchronization,definition.isReadOnly(), debugEnabled, null);
    }
}

private Object suspend(Object transaction) throws TransactionException {
    List suspendedSynchronizations = null;
    Object holder = doSuspend(transaction);
    // 如果有设置
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        // 执行TransactionSynchronization的钩子方法suspend
        suspendedSynchronizations = TransactionSynchronizationManager.getSynchronizations();
        for (Iterator it = suspendedSynchronizations.iterator(); it.hasNext();) {
            ((TransactionSynchronization) it.next()).suspend();
        }
        // 清除当前事务的Synchronizations
        TransactionSynchronizationManager.clearSynchronization();
    }
    // 简单的承载类
    return new SuspendedResourcesHolder(suspendedSynchronizations, holder);
}

private TransactionStatus newTransactionStatus(Object transaction, boolean newTransaction,boolean newSynchronization, boolean readOnly, boolean debug, Object suspendedResources) {
    // 标识为true且当前线程Synchronization出于未激活状态
    boolean actualNewSynchronization = newSynchronization &&
        !TransactionSynchronizationManager.isSynchronizationActive();
    if (actualNewSynchronization) {
        TransactionSynchronizationManager.initSynchronization();
    }
    return new DefaultTransactionStatus(transaction, newTransaction, actualNewSynchronization, readOnly, debug, suspendedResources);
    
```

由子类`DataSourceTransactionManager`实现的抽象方法

```java
protected Object doGetTransaction() {
    // 判断当前线程是否开启过事务
    if (TransactionSynchronizationManager.hasResource(this.dataSource)) {
        // 获取当前已存在的 connectoin holder
        ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
        return new DataSourceTransactionObject(holder);
    }
    else {
        // doBegin会赋值holder
        return new DataSourceTransactionObject();
    }
}

protected Object doSuspend(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    txObject.setConnectionHolder(null);
    // 移除resources中的resource对象
    return TransactionSynchronizationManager.unbindResource(this.dataSource);
}

protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    // 判断 connection holder 是否为空
	// 两种场景下可能为空：
	// 1. 上下文不存在事务的时候
	// 2. 上下文已存在的事务被挂起的时候
    if (txObject.getConnectionHolder() == null) {
        // 开启新的 connection
        Connection con = DataSourceUtils.getConnection(this.dataSource, false);
        // 绑定到事务对象transaction中
        txObject.setConnectionHolder(new ConnectionHolder(con));
    }

    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        // apply read-only
        if (definition.isReadOnly()) {
            try {
                con.setReadOnly(true);
            }
            catch (Exception ex) {
                // SQLException or UnsupportedOperationException
            }
        }

        // 设置事务隔离级别
        // apply isolation level
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            txObject.setPreviousIsolationLevel(new Integer(con.getTransactionIsolation()));
            con.setTransactionIsolation(definition.getIsolationLevel());
        }

        // 若 connection 为自动提交则取消
        // Switch to manual commit if necessary. This is very expensive in some JDBC
        // drivers, so we don't want to do it unnecessarily (for example if we're configured
        // Commons DBCP to set it already)
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            con.setAutoCommit(false);
        }

        // 设置超时时间
        // register transaction timeout
        if (definition.getTimeout() != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(definition.getTimeout());
        }

        // 将当前 connection holder 绑定到当前上下文
        // bind the connection holder to the thread
        TransactionSynchronizationManager.bindResource(this.dataSource, txObject.getConnectionHolder());
    }
    catch (SQLException ex) {
        throw new CannotCreateTransactionException("Could not configure connection", ex);
    }
}
```

`DataSourceUtils`

```java
public static Connection getConnection(DataSource ds, boolean allowSynchronization)
	    throws CannotGetJdbcConnectionException {
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(ds);
    if (conHolder != null) {
        return conHolder.getConnection();
    }
    else {
        try {
            Connection con = ds.getConnection();
            if (allowSynchronization && TransactionSynchronizationManager.isSynchronizationActive()) {
                // use same Connection for further JDBC actions within the transaction
                // thread object will get removed by synchronization at transaction completion
                conHolder = new ConnectionHolder(con);
                TransactionSynchronizationManager.bindResource(ds, conHolder);
                // 注册回调
                TransactionSynchronizationManager.registerSynchronization(new ConnectionSynchronization(conHolder, ds));
            }
            return con;
        }
        catch (SQLException ex) {
            throw new CannotGetJdbcConnectionException("Could not get JDBC connection", ex);
        }
    }
}

private static class ConnectionSynchronization extends TransactionSynchronizationAdapter {

    private ConnectionHolder connectionHolder;
    private DataSource dataSource;

    private ConnectionSynchronization(ConnectionHolder connectionHolder, DataSource dataSource) {
        this.connectionHolder = connectionHolder;
        this.dataSource = dataSource;
    }

    public void suspend() {
        TransactionSynchronizationManager.unbindResource(this.dataSource);
    }

    public void resume() {
        TransactionSynchronizationManager.bindResource(this.dataSource, this.connectionHolder);
    }

    public void beforeCompletion() throws CannotCloseJdbcConnectionException {
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        closeConnectionIfNecessary(this.connectionHolder.getConnection(), this.dataSource);
    }
}
```

再来看`AbstractPlatformTransactionManager`的rollback方法

```java
public final void rollback(TransactionStatus status) throws TransactionException {
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    try {
        try {
            // 触发钩子方法beforeCompletion
            triggerBeforeCompletion(defStatus);
            if (status.isNewTransaction()) {
                // 直接触发回滚
                doRollback(defStatus);
            }
            // 有多个事务
            else if (defStatus.getTransaction() != null) {
                // 设置rollbackOnly标识
                doSetRollbackOnly(defStatus);
            }
            else {
				// log
            }
        }
        catch (TransactionException ex) {
            triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_UNKNOWN, ex);
            throw ex;
        }
        triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_ROLLED_BACK, null);
    }
    finally {
        cleanupAfterCompletion(defStatus);
    }
}

private void triggerAfterCompletion(DefaultTransactionStatus status, int completionStatus, Throwable ex) {
    if (status.isNewSynchronization()) {
        try {
            for (Iterator it = TransactionSynchronizationManager.getSynchronizations().iterator(); it.hasNext();) {
                TransactionSynchronization synchronization = (TransactionSynchronization) it.next();
                // 触发钩子方法
                synchronization.afterCompletion(completionStatus);
            }
        }
        catch (RuntimeException tsex) {
            throw tsex;
        }
        catch (Error tserr) {
            throw tserr;
        }
    }
}

private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    if (status.isNewSynchronization()) {
        // 清理Synchronization
        TransactionSynchronizationManager.clearSynchronization();
    }
    if (status.isNewTransaction()) {
        doCleanupAfterCompletion(status.getTransaction());
    }
    // 当存在挂起的事务时，执行恢复挂起的事务
    if (status.getSuspendedResources() != null) {
        resume(status.getTransaction(), status.getSuspendedResources());
    }
}

private void resume(Object transaction, Object suspendedResources) throws TransactionException {
    SuspendedResourcesHolder resourcesHolder = (SuspendedResourcesHolder) suspendedResources;
    if (resourcesHolder.getSuspendedSynchronizations() != null) {
        TransactionSynchronizationManager.initSynchronization();
        for (Iterator it = resourcesHolder.getSuspendedSynchronizations().iterator(); it.hasNext();) {
            TransactionSynchronization synchronization = (TransactionSynchronization) it.next();
            // 触发钩子方法
            synchronization.resume();
            TransactionSynchronizationManager.registerSynchronization(synchronization);
        }
    }
    // 绑定resource资源到当前线程
    doResume(transaction, resourcesHolder.getSuspendedResources());
}
```

在子类`DataSourceTransactionManager`实现的方法

```java
protected void doCleanupAfterCompletion(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    // remove the connection holder from the thread
    TransactionSynchronizationManager.unbindResource(this.dataSource);
    // reset connection
    Connection con = txObject.getConnectionHolder().getConnection();

    try {
        // reset to auto-commit
        if (txObject.getMustRestoreAutoCommit()) {
            con.setAutoCommit(true);
        }

        // reset transaction isolation to previous value, if changed for the transaction
        if (txObject.getPreviousIsolationLevel() != null) {
            con.setTransactionIsolation(txObject.getPreviousIsolationLevel().intValue());
        }

        // reset read-only
        if (con.isReadOnly()) {
            con.setReadOnly(false);
        }
    }
    catch (Exception ex) {
    }

    try {
        DataSourceUtils.closeConnectionIfNecessary(con, this.dataSource);
    }
    catch (CleanupFailureDataAccessException ex) {
    }
}
```

再来看`AbstractPlatformTransactionManager`的commit方法

```java
public final void commit(TransactionStatus status) throws TransactionException {
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    // 如果设置了rollbackOnly标识，在rollback方法中设置
    if (status.isRollbackOnly() ||
        (defStatus.getTransaction() != null && isRollbackOnly(defStatus.getTransaction()))) {
        rollback(status);
    }

    else {
        try {
            try {
                // 触发钩子方法
                triggerBeforeCommit(defStatus);
                triggerBeforeCompletion(defStatus);
                if (status.isNewTransaction()) {
                    // 直接提交
                    doCommit(defStatus);
                }
            }
            catch (UnexpectedRollbackException ex) {
                 // 触发钩子方法
                triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_ROLLED_BACK, ex);
                throw ex;
            }
            catch (TransactionException ex) {
                if (this.rollbackOnCommitFailure) {
                    doRollbackOnCommitException(defStatus, ex);
                     // 触发钩子方法
                    triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_ROLLED_BACK, ex);
                }
                else {
                     // 触发钩子方法
                    triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_UNKNOWN, ex);
                }
                throw ex;
            }
            catch (RuntimeException ex) {
                doRollbackOnCommitException(defStatus, ex);
                 // 触发钩子方法
                triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_ROLLED_BACK, ex);
                throw ex;
            }
            catch (Error err) {
                doRollbackOnCommitException(defStatus, err);
                 // 触发钩子方法
                triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_UNKNOWN, err);
                throw err;
            }
             // 触发钩子方法
            triggerAfterCompletion(defStatus, TransactionSynchronization.STATUS_COMMITTED, null);
        }
        finally {
             // 触发钩子方法
            cleanupAfterCompletion(defStatus);
        }
    }
}
```

我们手动模拟TransactionSynchronization的回调

```java
import org.springframework.transaction.support.TransactionSynchronizationAdapter;
import org.springframework.transaction.support.TransactionSynchronizationManager;
public class MyTransactionSynchronizationAdapter extends TransactionSynchronizationAdapter {
    private MySessionFactory mySessionFactory;
    public MyTransactionSynchronizationAdapter(MySessionFactory mySessionFactory) {
        this.mySessionFactory = mySessionFactory;
    }

    @Override
    public void beforeCommit(boolean readOnly) {
        if (!readOnly) {
            MySession mySession = (MySession) TransactionSynchronizationManager.getResource(mySessionFactory);
            mySession.beginTransaction();
        }
    }

    @Override
    public void afterCompletion(int status) {
        MySession mySession = (MySession) TransactionSynchronizationManager.getResource(mySessionFactory);
        if (STATUS_COMMITTED == status) {
            mySession.commit();
        }
    }
} 

import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;
public class MySessionFactory {
    public MySession getSession() {
        if (TransactionSynchronizationManager.hasResource(this)) {
            return getCurrentSession();
        }
        return openSession();
    }

    private MySession openSession() {
        MySession mySession = new MySession();
        TransactionSynchronization transactionSynchronization = new MyTransactionSynchronizationAdapter(this);
        //已在获取TransactionStatus(AbstractPlatformTransactionManager.getTransaction)中initSynchronization
   TransactionSynchronizationManager.registerSynchronization(transactionSynchronization);
        TransactionSynchronizationManager.bindResource(this, mySession);
        return mySession;
    }

    private MySession getCurrentSession() {
        return (MySession) TransactionSynchronizationManager.getResource(this);
    }
}  

public class MySession {
    public void save() {
        System.out.println("save");
    }

    public void beginTransaction() {
        System.out.println("beginTransaction");
    }

    public void commit() {
        System.out.println("commit");
    }
}  

public class ExampleService {
    private MySessionFactory mySessionFactory;

    public void setMySessionFactory(MySessionFactory mySessionFactory) {
        this.mySessionFactory = mySessionFactory;
    }

    public void save() {
        mySessionFactory.getSession().save();
    }
}

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class SynchronizationMain {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("transactionalBeanFactory.xml");
        ExampleService proxyFactory5 = (ExampleService) context.getBean("proxyFactory5", ExampleService.class);
        proxyFactory5.save();
    }
}
```

`transactionalBeanFactory.xml`为

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>
    <!-- 配置数据源 -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
        <property name="driverClassName">
            <value>com.mysql.jdbc.Driver</value>
        </property>
        <property name="url">
            <value>urlt</value>
        </property>
        <property name="username">
            <value>username</value>
        </property>
        <property name="password">
            <value>password</value>
        </property>
    </bean>

    <!-- 配置事务管理 -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource">
            <ref bean="dataSource"></ref>
        </property>
    </bean>

    <bean id="mySessionFactory" class="com.youxiao.doc.synchronization.MySessionFactory"/>

    <bean id="serviceTarget" class="com.youxiao.doc.synchronization.ExampleService">
        <property name="mySessionFactory">
            <ref bean="mySessionFactory"/>
        </property>
    </bean>

    <bean id="proxyFactory5" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
        <property name="transactionManager">
            <ref local="transactionManager"/>
        </property>
        <property name="target">
            <ref local="serviceTarget"/>
        </property>
        <property name="proxyTargetClass">
            <value>true</value>
        </property>
        <property name="transactionAttributes">
            <props>
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>
</beans>
```

其输出为

save
beginTransaction
commit

意味着，通过TransactionSynchronizationManager的registerSynchronization方法可以主动注册自定义TransactionSynchronization回调，通过TransactionSynchronizationManager的bindResource方法可以主动绑定任意资源对象到其中。而调用的时机由Spring在事务的不同阶段控制，如前述所示。
