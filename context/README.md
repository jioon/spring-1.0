spring-context

从最基本的使用开始。

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class ContextMain {

    public static void main(String[] args) {
        System.setProperty("spring", "classpath");
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        Object bean = context.getBean("exampleBean", ExampleBean.class);
        ((ExampleBean) bean).name();
        System.out.println(((ExampleBean) bean).getResource().getFile().getAbsolutePath());
    }
}

public class ExampleBean {
    private String name;
    private Resource resource;

    public void name() {
        System.out.println("I am " + name + " Bean!");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Resource getResource() {
        return resource;
    }

    public void setResource(Resource resource) {
        this.resource = resource;
    }
}

import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValue;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
public class ExampleBeanFactoryPostProcessorBean implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        BeanDefinition beanDefinition = configurableListableBeanFactory.getBeanDefinition("exampleBean");
        System.out.println("before postProcessBeanFactory : " + beanDefinition.getPropertyValues().getPropertyValue("name"));
        PropertyValue[] values = beanDefinition.getPropertyValues().getPropertyValues();
        for (int i = 0; i < values.length; i++) {
            PropertyValue value = values[i];
            PropertyValue newValue = new PropertyValue("name", "NewName");
            if ("name".equals(value.getName())) {
                beanDefinition.getPropertyValues().setPropertyValueAt(newValue, i);
            }
        }
        System.out.println("after postProcessBeanFactory : " + beanDefinition.getPropertyValues().getPropertyValue("name"));
    }
}

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
public class ExampleBeanPostProcessorBean implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) throws BeansException {
        if ("exampleBean".equals(name)) {
            System.out.println("postProcessBeforeInitialization : " + ((ExampleBean) bean).getName());
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String name) throws BeansException {
        if ("exampleBean".equals(name)) {
            System.out.println("postProcessAfterInitialization change name 2 BeanPostProcessorName");
            ((ExampleBean) bean).setName("BeanPostProcessorName");
        }
        return bean;
    }
}
```

beans.xml为

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>
    <bean id="exampleBean" class="com.youxiao.doc.context.ExampleBean">
        <property name="name">
            <value>Example</value>
        </property>
        <property name="resource">
            <value>${spring}:beans.xml</value>
        </property>
    </bean>
    <bean id="exampleBeanPostProcessorBean" class="com.youxiao.doc.context.ExampleBeanPostProcessorBean"/>
    <bean id="exampleBeanFactoryPostProcessorBean" class="com.youxiao.doc.context.ExampleBeanFactoryPostProcessorBean"/>
</beans>
```

ClassPathXmlApplicationContext使用方式和XmlBeanFactory基本及一致，这和applicationContext设计有关系，applicationContext可以看做是beanFactory的高级版本,他们的主要区别是：

>The basis for the context package is the ApplicationContext interface, located in the
>org.springframework.context package. First of all it provides all the functionality the BeanFactory also
>provides. To be able to work in a more framework-oriented way, using layering and hierarchical contexts, the
>context package also provides the following:
>• MessageSource, providing access to messages in, i18n-style
>• Access to resources, like URLs and files
>• Event propagation to beans implementing the ApplicationListener interface
>• Loading of multiple contexts, allowing some of them to be focused and used for example only by the
>web-layer of an application

![ClassPathXmlApplicationContext](https://github.com/jioon/spring-1.0/blob/master/context/image/ClassPathXmlApplicationContext.jpg?raw=true)

我们从ClassPathXmlApplicationContext的实例化过程来一窥究竟。

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this.configLocations = new String[] {configLocation};
    refresh();
}
```

refresh()方法是applicationContext的核心加载方法,从1.x版本到5.x版本都被沿用，而功能在不断扩展,其默认实现在AbstractApplicationContext类中

```java
public void refresh() throws BeansException {
    this.startupTime = System.currentTimeMillis();

    // 1, refreshBeanFactory
    // tell subclass to refresh the internal bean factory
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();

    // 2, ContextResourceEditor
    // configure the bean factory with context semantics
    beanFactory.registerCustomEditor(Resource.class, new ContextResourceEditor(this));
    // 使用了beanFactory的beanPostProcessor扩展点
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyType(ResourceLoader.class);
    beanFactory.ignoreDependencyType(ApplicationContext.class);
    // beanFactory的扩展点
    postProcessBeanFactory(beanFactory);

    // invoke factory processors registered with the context instance
    for (Iterator it = getBeanFactoryPostProcessors().iterator(); it.hasNext();) {
        BeanFactoryPostProcessor factoryProcessor = (BeanFactoryPostProcessor) it.next();
        // 通过context.addBeanFactoryPostProcessor 添加的ProcessBeanFactory实例
        // 默认为空
        factoryProcessor.postProcessBeanFactory(beanFactory);
    }

    // 3,invokeBeanFactoryPostProcessors
    // invoke factory processors registered as beans in the context
    invokeBeanFactoryPostProcessors();

    // 4,registerBeanPostProcessors
    // register bean processor that intercept bean creation
    registerBeanPostProcessors();

    // 5,initMessageSource
    // initialize message source for this context
    initMessageSource();

    // 默认空实现
    // initialize other special beans in specific context subclasses
    onRefresh();

    // 6,refreshListeners
    // check for listener beans and register them
    refreshListeners();

    // 7,preInstantiateSingletons
    // instantiate singletons this late to allow them to access the message source
    beanFactory.preInstantiateSingletons();

    // last step: publish respective event
    publishEvent(new ContextRefreshedEvent(this));
}
```

一步步看， 

1、refreshBeanFactory，是一个抽象方法，交由子类实现，

```java
protected abstract void refreshBeanFactory() throws BeansException;
```

AbstractXmlApplicationContext对其做了实现

```java
protected void refreshBeanFactory() throws BeansException {
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        initBeanDefinitionReader(beanDefinitionReader);
        loadBeanDefinitions(beanDefinitionReader);
        this.beanFactory = beanFactory;
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing XML document for application context [" + getDisplayName() + "]", ex);
    } 
}
```

这和直接使用XmlBeanFactory基本一致，

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 被ClassPathXmlApplicationContext重载
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        for (int i = 0; i < configLocations.length; i++) {
            // XmlBeanFactory 构造方法入参就是Resource
            reader.loadBeanDefinitions(getResource(configLocations[i]));
        }
    }
}
```

此时getResource来自DefaultResourceLoader，其实现了ResourceLoader接口

```java
public interface ResourceLoader {

	/** Pseudo URL prefix for loading from the class path */
	String CLASSPATH_URL_PREFIX = "classpath:";

	/**
	 * Return a Resource handle for the specified resource.
	 * The handle should always be a reusable resource descriptor,
	 * allowing for multiple getInputStream calls.
	 * <p><ul>
	 * <li>Must support fully qualified URLs, e.g. "file:C:/test.dat".
	 * <li>Must support classpath pseudo-URLs, e.g. "classpath:test.dat".
	 * <li>Should support relative file paths, e.g. "WEB-INF/test.dat".
	 * (This will be implementation-specific, typically provided by an
	 * ApplicationContext implementation.)
	 * </ul>
	 * <p>Note that a Resource handle does not imply an existing resource;
	 * you need to invoke Resource's "exists" to check for existence.
	 * @param location resource location
	 * @return Resource handle
	 * @see #CLASSPATH_URL_PREFIX
	 * @see org.springframework.core.io.Resource#exists
	 * @see org.springframework.core.io.Resource#getInputStream
	 */
	Resource getResource(String location);
}

public Resource getResource(String location) {
    if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()));
    }
    else {
        try {
            // try URL
            URL url = new URL(location);
            return new UrlResource(url);
        }
        catch (MalformedURLException ex) {
            // no URL -> resolve resource path
            return getResourceByPath(location);
        }
    }
}

/**
* Return a Resource handle for the resource at the given path.
* <p>Default implementation supports class path locations. This should
* be appropriate for standalone implementations but can be overridden,
* e.g. for implementations targeted at a Servlet container.
* @param path path to the resource
* @return Resource handle
* @see ClassPathResource
* @see FileSystemXmlApplicationContext#getResourceByPath
* @see XmlWebApplicationContext#getResourceByPath
*/
protected Resource getResourceByPath(String path) {
    return new ClassPathResource(path);
}
```

DefaultResourceLoader默认path在classpth目录下，当然可以重载getResourceByPath方法替换其实现，例如FileSystemXmlApplicationContext中就重载了其实现。

2、ContextResourceEditor继承了ResourceEditor，实质就是识别使用了${}的资源地址，例如前述例子中ExampleBean的resource成员，其在beans.xml中配置的是

```xml
<property name="resource">
    <value>${spring}:beans.xml</value>
</property>
```

实际注入的是classpath:bean.xml，过程是ResourceEditor将${spring}:beans.xml转换成了classpath:beans.xml，交由applicationContext的ResourceLoader加载了classpath下的beans.xml文件，变身成Resource对象注入到ExampleBean的成员resource中。

3、invokeBeanFactoryPostProcessors

```java
private void invokeBeanFactoryPostProcessors() throws BeansException {
    // BeanFactory的getBeanNamesForType方法获取容器中BeanFactoryPostProcessor BeanDefinition的name数组，之后逐一调用getBean方法获取BeanFactoryPostProcessor bean
    String[] beanNames = getBeanDefinitionNames(BeanFactoryPostProcessor.class);
    BeanFactoryPostProcessor[] factoryProcessors = new BeanFactoryPostProcessor[beanNames.length];
    for (int i = 0; i < beanNames.length; i++) {
        factoryProcessors[i] = (BeanFactoryPostProcessor) getBean(beanNames[i]);
    }
    // 按order顺序调用
    Arrays.sort(factoryProcessors, new OrderComparator());
    for (int i = 0; i < factoryProcessors.length; i++) {
        BeanFactoryPostProcessor factoryProcessor = factoryProcessors[i];
        factoryProcessor.postProcessBeanFactory(getBeanFactory());
    }
}
```

4、registerBeanPostProcessors

```java
private void registerBeanPostProcessors() throws BeansException {
    String[] beanNames = getBeanDefinitionNames(BeanPostProcessor.class);
    if (beanNames.length > 0) {
        List beanProcessors = new ArrayList();
        for (int i = 0; i < beanNames.length; i++) {
            beanProcessors.add(getBean(beanNames[i]));
        }
        // 按order顺序排序
        Collections.sort(beanProcessors, new OrderComparator());
        for (Iterator it = beanProcessors.iterator(); it.hasNext();) {
            // 调用时机在AbstractAutowireCapableBeanFactory createBean中
            getBeanFactory().addBeanPostProcessor((BeanPostProcessor) it.next());
        }
    }
}
```

BeanFactoryPostProcessor、BeanPostProcessor名字命名非常相近，接口的设计也非常相近，只是调用的时机不一致。

BeanFactoryPostProcessor可以与bean定义交互，并修改bean定义，但不应该与bean实例交互，这样做可能会导致bean过早实例化，违反容器并导致意外的副作用。如果需要bean实例交互，就可以使用BeanPostProcessor。

```java
public interface BeanFactoryPostProcessor {
     //参数是beanFactory实例，可通过实例修改容器中的BeanDefinition
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

// aop通过此扩展点完成
public interface BeanPostProcessor {
    //实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务  
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    //实例化、依赖注入、初始化完毕时执行  
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

运行文章开头的例子，观察日志输出：

before postProcessBeanFactory : PropertyValue: name='name'; value=[Example]
after postProcessBeanFactory : PropertyValue: name='name'; value=[NewName]
postProcessBeforeInitialization : NewName
postProcessAfterInitialization change name 2 BeanPostProcessorName
I am BeanPostProcessorName Bean!

和预期的调用时机一致，

BeanFactoryPostProcessor->postProcessBeforeInitialization ->postProcessAfterInitialization 

再来看spring内置的CustomEditorConfigurer，此类是PropertyEditor的容器，通过属性customEditors承载，该类实现了BeanFactoryPostProcessor, Ordered接口。其使用方式如下：

```java
import java.beans.PropertyEditorSupport;
import java.text.SimpleDateFormat;
import java.util.Date;

public class DatePropertyEditor extends PropertyEditorSupport {
    private String format;

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        try {
            SimpleDateFormat sdf = new SimpleDateFormat(this.format);
            Date date = sdf.parse(text);
            this.setValue(date);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}  
```

beans.xml加入此项配置

```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="java.util.Date">
                <bean class="com.youxiao.doc.context.DatePropertyEditor">
                    <property name="format">
                        <value>yyyy-MM-dd</value>
                    </property>
                </bean>
            </entry>
        </map>
    </property>
</bean>
```

CustomEditorConfigurer的实现如下：

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    if (this.customEditors != null) {
        for (Iterator it = this.customEditors.keySet().iterator(); it.hasNext();) {
            Object key = it.next();
            Class requiredType = null;
            if (key instanceof Class) {
                requiredType = (Class) key;
            }
            else if (key instanceof String) {
                String className = (String) key;
                try {
                    requiredType = Class.forName(className, true, Thread.currentThread().getContextClassLoader());
                }
                catch (ClassNotFoundException ex) {
                    throw new BeanInitializationException("Could not load required type [" + className + "] for custom editor", ex);
                }
            }
            else {
                throw new BeanInitializationException("Invalid key [" + key + "] for custom editor - " + "needs to be Class or String");
            }
            Object value = this.customEditors.get(key);
            if (!(value instanceof PropertyEditor)) {
                throw new BeanInitializationException("Mapped value for custom editor is not of type " + "java.beans.PropertyEditor");
            }
            // 注册到容器的customEditor中
            beanFactory.registerCustomEditor(requiredType, (PropertyEditor) value);
        }
    }
}
```

5、initMessageSource

用以支持spring国际化，可查考[Spring 国际化信息](https://www.iteye.com/blog/stamen-1541732)

6、refreshListeners

用以支持spring事件驱动

```java
private void refreshListeners() throws BeansException {
    Collection listeners = getBeansOfType(ApplicationListener.class, true, false).values();
    for (Iterator it = listeners.iterator(); it.hasNext();) {
        ApplicationListener listener = (ApplicationListener) it.next();
        // 简单加入到聚合类ApplicationEventMulticaster中
        addListener(listener);
    }
}
```

结合后续的publishEvent一起

```java
public void publishEvent(ApplicationEvent event) {
    this.eventMulticaster.onApplicationEvent(event);
    if (this.parent != null) {
        parent.publishEvent(event);
    }
}
```

很明显，当特定event发生时，使用onApplicationEvent方法触发监听者处理。

用ContextRefreshedEvent来观察event

```java
public class ContextRefreshedEvent extends ApplicationEvent {
	public ContextRefreshedEvent(ApplicationContext source) {
		super(source);
	}

	public ApplicationContext getApplicationContext() {
		return (ApplicationContext) getSource();
	}
}

public abstract class ApplicationEvent extends EventObject {

	/** System time when the event happened */
	private long timestamp;

	/**
	 * Creates a new ApplicationEvent.
	 * @param source component that published the event
	 */
	public ApplicationEvent(Object source) {
		super(source);
		timestamp = System.currentTimeMillis();
	}
	public long getTimestamp() {
		return timestamp;
	}
}

package java.util;
import java.io.Serializable;
public class EventObject implements Serializable {
    private static final long serialVersionUID = 5516075349620653480L;
    protected transient Object source;

    public EventObject(Object source) {
        if (source == null) {
            throw new IllegalArgumentException("null source");
        } else {
            this.source = source;
        }
    }

    public Object getSource() {
        return this.source;
    }

    public String toString() {
        return this.getClass().getName() + "[source=" + this.source + "]";
    }
}
```

EventObject为jdk中事件类，所有事件都从其派生。构造时时，source对象为最初发生 Event 的对象。

再看事件承载类ApplicationEventMulticasterImpl

```java
public class ApplicationEventMulticasterImpl implements ApplicationEventMulticaster {
	/** Set of listeners */
	private Set eventListeners = new HashSet();
	public void addApplicationListener(ApplicationListener l) {
		eventListeners.add(l);
	}

	public void removeApplicationListener(ApplicationListener l) {
		eventListeners.remove(l);
	}

    // 触发监听者处理事件
	public void onApplicationEvent(ApplicationEvent e) {
		Iterator i = eventListeners.iterator();
		while (i.hasNext()) {
			ApplicationListener l = (ApplicationListener) i.next();
			l.onApplicationEvent(e);
		}
	}

	public void removeAllListeners() {
		eventListeners.clear();
	}
}

public interface ApplicationEventMulticaster extends ApplicationListener {
	void addApplicationListener(ApplicationListener listener);
	void removeApplicationListener(ApplicationListener listener);
	void removeAllListeners();
}

public interface ApplicationListener extends EventListener {
    void onApplicationEvent(ApplicationEvent e);
}

package java.util;
public interface EventListener {
}
```

EventListener是一个标记类接口，实现了该接口就是事件监听者。

看一个简单的例子

```java
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ApplicationEventMulticaster;
import org.springframework.context.event.ApplicationEventMulticasterImpl;

public class EventExample {
    private ApplicationEventMulticaster eventMulticaster = new ApplicationEventMulticasterImpl();

    static class WorkListener implements ApplicationListener {
        @Override
        public void onApplicationEvent(ApplicationEvent applicationEvent) {
            if (applicationEvent instanceof WorkIn) {
                System.out.println("上班了！");
            }
            if (applicationEvent instanceof WorkOut) {
                System.out.println("下班了！");
            }
        }
    }

    static class Work {
    }

    static class WorkIn extends ApplicationEvent {
        public WorkIn(Work source) {
            super(source);
        }

        public WorkIn getWorkIn() {
            return (WorkIn) this.getSource();
        }
    }

    static class WorkOut extends ApplicationEvent {
        public WorkOut(Work source) {
            super(source);
        }

        public WorkOut getWorkOut() {
            return (WorkOut) this.getSource();
        }
    }

    public static void main(String[] args) {
        EventExample example = new EventExample();
        example.eventMulticaster.addApplicationListener(new WorkListener());
        Work work = new Work();
        example.eventMulticaster.onApplicationEvent(new WorkIn(work));
        System.out.println("工作中！！！！");
        example.eventMulticaster.onApplicationEvent(new WorkOut(work));
    }
}
```



7、preInstantiateSingletons，来自ConfigurableListableBeanFactory接口，其默认实现在DefaultListableBeanFactory中，实质就是对容器中单例且非延迟加载的类型进行实例化。

```

/**
* bean 初始化过程如下:
* 1 : bean 构造初始化
* 2 : bean 属性注入 (通过 bean definition 中的 property , autowire(byType, byName) 实现)
* 3 : bean 若实现 BeanNameAware 接口，调用 setBeanName() 方法
* 4 : bean 若实现 BeanFactoryAware 接口, 调用 setBeanFactory() 方法
* 5 : 遍历调用 BeanFactory 中注册的 BeanPostProcessor 实例的 postProcessorBeforeInitialization() 方法
* 6 : bean 若实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
* 7 : bean 实例对应的 bean definition 中若定义了 init-method 属性则调用对应的 init 方法
* 8 : 遍历调用 BeanFactory 中注册的 BeanPostProcessor 实例的 postProcessorAfterInitialization() 方法
*/
/**
* Ensure that all non-lazy-init singletons are instantiated, also considering
* FactoryBeans. Typically invoked at the end of factory setup, if desired.
*/
public void preInstantiateSingletons() {
	for (Iterator it = this.beanDefinitionNames.iterator(); it.hasNext();) {
		String beanName = (String) it.next();
		if (containsBeanDefinition(beanName)) {
			RootBeanDefinition bd = getMergedBeanDefinition(beanName, false);
			if (bd.isSingleton() && !bd.isLazyInit()) {
				if (FactoryBean.class.isAssignableFrom(bd.getBeanClass())) {
					FactoryBean factory = (FactoryBean) getBean(FACTORY_BEAN_PREFIX + beanName);
					if (factory.isSingleton()) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
	}
}
```
