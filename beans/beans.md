Spring-beans

从最基本的使用开始.

```java
import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.core.io.ClassPathResource;

public class Main {
    public static void main(String[] args) {
        XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
        Object bean = factory.getBean("exampleBean", ExampleBean.class);
        ((ExampleBean) bean).name();
    }
}

public class ExampleBean {
    public void name() {
        System.out.println("I am ExampleBean!");
    }
}
```

beans.xml为

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">

<beans>
   <bean id="exampleBean" class="com.youxiao.doc.beans.ExampleBean" />
</beans>
```

ClassPathResource的继承结构如下,其代表了classpath下的资源.

![ClassPathResource 继承结构](https://github.com/jioon/spring-1.0/blob/master/beans/image/ClassPathResource.jpg?raw=true)



Resource接口从1.0演进到最新的5.x,只增加了少量的功能,而实现更加精妙.spring面向接口的设计做的非常完备,值得学习.

```java
public interface InputStreamSource {
    // 返回的是jdk的InputStream
	InputStream getInputStream() throws IOException;
}

public interface Resource extends InputStreamSource {
	boolean exists();
	boolean isOpen();
	URL getURL() throws IOException;
	File getFile() throws IOException;
	String getDescription();
}
```

除了上述的ClassPathResource外,还有FileSystemResource InputStreamResource UrlResource,代表了不同的resouce能力.

再来看XmlBeanFactory,作为核心接口实现,其设计就复杂得多.

继承体系中,每一个都有特殊的用途,例如BeanFactory提供getBean containsBean等容器核心能力,ListableBeanFactory提供getBeansOfType getBeanDefinitionNames等容器数据列表能力,HierarchicalBeanFactory提供getParentBeanFactory感知父类容器能力,ConfigurableBeanFactory提供registerCustomEditor ignoreDependencyType等操作容器的能力,AutowireCapableBeanFactory提供autowire autowireBeanProperties等bean自动注入相关能力.

![XmlBeanFactory](https://github.com/jioon/spring-1.0/blob/master/beans/image/XmlBeanFactory.jpg?raw=true)

跟随启动代码一步步向下看

```java
public XmlBeanFactory(Resource resource) throws BeansException {
    this(resource, null);
}

public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
    super(parentBeanFactory);
    this.reader.loadBeanDefinitions(resource);
}

public AbstractBeanFactory(BeanFactory parentBeanFactory) {
    // 此时parentBeanFactory为null
    this();
    this.parentBeanFactory = parentBeanFactory;
}
public AbstractBeanFactory() {
    ignoreDependencyType(BeanFactory.class);
}
public void ignoreDependencyType(Class type) {
    this.ignoreDependencyTypes.add(type);
}
```

XmlBeanDefinitionReader 中

```java
public void loadBeanDefinitions(Resource resource) throws BeansException {
    // 以免篇幅太长,省略日志 部分异常相关代码,后续不再赘述
    // spring 日志 和 异常 处理是教科书级别,值得借鉴
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    // 默认需要验证,可通过setValidating更改
    factory.setValidating(this.validating);
    DocumentBuilder docBuilder = factory.newDocumentBuilder();
    docBuilder.setErrorHandler(new BeansErrorHandler());
    // 配置dtd资源解析器,EntityResolver维护了url->location的路径,1.0中只支持dtd
    // 默认为BeansDtdResolver,可通过setEntityResolver更改,例如有特殊的路径需要处理
    docBuilder.setEntityResolver(this.entityResolver != null ? this.entityResolver : new BeansDtdResolver());
    is = resource.getInputStream();
    Document doc = docBuilder.parse(is);
    registerBeanDefinitions(doc, resource);
}
```

如果SAX应用程序需要实现自定义处理外部实体,则必须实现EntityResolver接口,并使用setEntityResolver方法向SAX 驱动器注册一个实例.也就是说,对于解析一个xml,sax首先会读取该xml文档上的声明,根据声明去寻找相应的dtd定义,以便对文档的进行验证,默认的寻找规则,(即:通过网络,实现上就是声明DTD的地址URI地址来下载DTD声明),并进行认证,下载的过程是一个漫长的过程,而且当网络不可用时,这里会报错,就是应为相应的dtd没找到.
EntityResolver 由程序来实现寻找DTD/XSD声明的过程,比如我们将DTD/XSD放在项目的某处在实现时直接将此文档读取并返回个SAX即可,这样就避免了通过网络来寻找DTD/XSD的声明

```java
public interface EntityResolver {  
public abstract InputSource resolveEntity (String publicId,String systemId) throws SAXException, IOException;
}
```

例如

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC  "-//SPRING//DTD BEAN//EN"  "http://www.springframework.org/dtd/spring-beans.dtd">
<beans></beans>
```

其中publicId=-//SPRING//DTD BEAN//EN

systemId=http://www.springframework.org/dtd/spring-beans.dtd

BeansDtdResolver实际处理方式：

```java
public InputSource resolveEntity(String publicId, String systemId) throws IOException {
    if (systemId != null && systemId.indexOf(DTD_NAME) > systemId.lastIndexOf("/")) {
        String dtdFile = systemId.substring(systemId.indexOf(DTD_NAME));
        try {
            // path最终为,BeansDtdResolver classpath路径下
            // /org/springframework/beans/factory/xml/spring-beans.dtd
            // BeansDtdResolver和spring-beans.dtd在同一路径
            Resource resource = new ClassPathResource(SEARCH_PACKAGE + dtdFile, getClass());
            InputSource source = new InputSource(resource.getInputStream());
            source.setPublicId(publicId);
            source.setSystemId(systemId);
            return source;
        }
        catch (IOException ex) {
         // log
        }
    }
    // use the default behaviour -> download from website or wherever
    return null;
}
```

回过头,查看XmlBeanDefinitionReader中

```java
  registerBeanDefinitions(doc, resource);
```

```java
public void registerBeanDefinitions(Document doc, Resource resource) throws BeansException {
    // 默认为DefaultXmlBeanDefinitionParser.class,可通过setParserClass更改,必须实现XmlBeanDefinitionParser接口
    XmlBeanDefinitionParser parser = (XmlBeanDefinitionParser) BeanUtils.instantiateClass(this.parserClass);
    // 委托给XmlBeanDefinitionParser解析xml
    parser.registerBeanDefinitions(getBeanFactory(), getBeanClassLoader(), doc, resource);
}
```

XmlBeanDefinitionParser接口只有一个方法

```java
public interface XmlBeanDefinitionParser {

	/**
	 * Parse bean definitions from the given DOM document,
	 * and register them with the given bean factory.
	 * @param beanFactory the bean factory to register the bean definitions with
	 * @param beanClassLoader class loader to use for bean classes
	 * (null suggests to not load bean classes but just register bean definitions
	 * with class names, for example when just registering beans in a registry
	 * but not actually instantiating them in a factory)
	 * @param doc the DOM document
	 * @param resource descriptor of the original XML resource
	 * (useful for displaying parse errors)
	 * @throws BeansException in case of parsing errors
	 */
	void registerBeanDefinitions(BeanDefinitionRegistry beanFactory, ClassLoader beanClassLoader, Document doc, Resource resource) throws BeansException;
}

```

其默认实现类为DefaultXmlBeanDefinitionParser,也是xml->beanFactory的实际入口.

该类中静态变量就指代spring-bean的一种配置,

```java
public static final String BEAN_NAME_DELIMITERS = ",; ";

/**
* Value of a T/F attribute that represents true.
* Anything else represents false. Case seNsItive.
*/
public static final String TRUE_VALUE = "true";
public static final String DEFAULT_VALUE = "default";

public static final String DEFAULT_LAZY_INIT_ATTRIBUTE = "default-lazy-init";
public static final String DEFAULT_DEPENDENCY_CHECK_ATTRIBUTE = "default-dependency-check";
public static final String DEFAULT_AUTOWIRE_ATTRIBUTE = "default-autowire";

public static final String BEAN_ELEMENT = "bean";
public static final String DESCRIPTION_ELEMENT = "description";
public static final String CLASS_ATTRIBUTE = "class";
public static final String PARENT_ATTRIBUTE = "parent";
public static final String ID_ATTRIBUTE = "id";
public static final String NAME_ATTRIBUTE = "name";
public static final String SINGLETON_ATTRIBUTE = "singleton";
public static final String DEPENDS_ON_ATTRIBUTE = "depends-on";
public static final String INIT_METHOD_ATTRIBUTE = "init-method";
public static final String DESTROY_METHOD_ATTRIBUTE = "destroy-method";
public static final String CONSTRUCTOR_ARG_ELEMENT = "constructor-arg";
public static final String INDEX_ATTRIBUTE = "index";
public static final String TYPE_ATTRIBUTE = "type";
public static final String PROPERTY_ELEMENT = "property";
public static final String REF_ELEMENT = "ref";
public static final String IDREF_ELEMENT = "idref";
public static final String BEAN_REF_ATTRIBUTE = "bean";
public static final String LOCAL_REF_ATTRIBUTE = "local";
public static final String LIST_ELEMENT = "list";
public static final String SET_ELEMENT = "set";
public static final String MAP_ELEMENT = "map";
public static final String KEY_ATTRIBUTE = "key";
public static final String ENTRY_ELEMENT = "entry";
public static final String VALUE_ELEMENT = "value";
public static final String NULL_ELEMENT = "null";
public static final String PROPS_ELEMENT = "props";
public static final String PROP_ELEMENT = "prop";

public static final String LAZY_INIT_ATTRIBUTE = "lazy-init";

public static final String DEPENDENCY_CHECK_ATTRIBUTE = "dependency-check";
public static final String DEPENDENCY_CHECK_ALL_ATTRIBUTE_VALUE = "all";
public static final String DEPENDENCY_CHECK_SIMPLE_ATTRIBUTE_VALUE = "simple";
public static final String DEPENDENCY_CHECK_OBJECTS_ATTRIBUTE_VALUE = "objects";

public static final String AUTOWIRE_ATTRIBUTE = "autowire";
public static final String AUTOWIRE_BY_NAME_VALUE = "byName";
public static final String AUTOWIRE_BY_TYPE_VALUE = "byType";
public static final String AUTOWIRE_CONSTRUCTOR_VALUE = "constructor";
public static final String AUTOWIRE_AUTODETECT_VALUE = "autodetect";
```

```java
public void registerBeanDefinitions(BeanDefinitionRegistry beanFactory, ClassLoader beanClassLoader,Document doc, Resource resource) {
    this.beanFactory = beanFactory;
    this.beanClassLoader = beanClassLoader;
    this.resource = resource;

    Element root = doc.getDocumentElement();
	//default-lazy-init
    this.defaultLazyInit = root.getAttribute(DEFAULT_LAZY_INIT_ATTRIBUTE);
    //default-dependency-check
    this.defaultDependencyCheck = root.getAttribute(DEFAULT_DEPENDENCY_CHECK_ATTRIBUTE);
    //default-autowire
    this.defaultAutowire = root.getAttribute(DEFAULT_AUTOWIRE_ATTRIBUTE);

    NodeList nl = root.getChildNodes();
    int beanDefinitionCounter = 0;
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element && BEAN_ELEMENT.equals(node.getNodeName())) {
            // 解析bean标签
            beanDefinitionCounter++;
            loadBeanDefinition((Element) node);
        }
    }
}
```

defaultLazyInit,defaultDependencyCheck,defaultAutowire用于处理xml中，具体逻辑见后续

    <beans default-autowire="autodetect" default-dependency-check="all" default-lazy-init="true">
    ...
    </beans>
```java
protected void loadBeanDefinition(Element ele) {
    //id
    String id = ele.getAttribute(ID_ATTRIBUTE);
    //name
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    List aliases = new ArrayList();
    if (nameAttr != null && !"".equals(nameAttr)) {
        // 意味name可以配置多个,使用 , ; 空格 分割即可
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, BEAN_NAME_DELIMITERS, true, true);
        aliases.addAll(Arrays.asList(nameArr));
    }

    if (id == null || "".equals(id) && !aliases.isEmpty()) {
        // id为空时且aliase不空时,使用第一个name作为id
        id = (String) aliases.remove(0);
    }

    // 核心 获取beanDefinition
    // 由于parseBeanDefinition为protected方法,可能会被重载,因此用AbstractBeanDefinition承接
    AbstractBeanDefinition beanDefinition = parseBeanDefinition(ele, id);

    if (id == null || "".equals(id)) {
        if (beanDefinition instanceof RootBeanDefinition) {
            // 不指定id name也可以正常工作的逻辑
            id = ((RootBeanDefinition) beanDefinition).getBeanClassName();
        }
        else {
            throw new BeanDefinitionStoreException(this.resource, "", "Child bean definition has neither 'id' nor 'name'");
        }
    }

    this.beanFactory.registerBeanDefinition(id, beanDefinition);
    for (Iterator it = aliases.iterator(); it.hasNext();) {
        this.beanFactory.registerAlias(id, (String) it.next());
    }
}
```

```java
protected AbstractBeanDefinition parseBeanDefinition(Element ele, String beanName) {
    String className = null;
    try {
        if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
            className = ele.getAttribute(CLASS_ATTRIBUTE);
        }
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }
        if (className == null && parent == null) {
            // class parent至少要有一个
            throw new BeanDefinitionStoreException(this.resource, beanName, "Either 'class' or 'parent' is required");
        }

        AbstractBeanDefinition bd = null;
        // property子属性解析
        MutablePropertyValues pvs = getPropertyValueSubElements(beanName, ele);

        if (className != null) {
            // 构造器方法解析
            ConstructorArgumentValues cargs = getConstructorArgSubElements(beanName, ele);
            RootBeanDefinition rbd = null;
            // 若beanClassLoader为空不装载class只注册className,见XmlBeanDefinitionParser注释,			 // 例如bean不需要实例化
            if (this.beanClassLoader != null) {
                Class clazz = Class.forName(className, true, this.beanClassLoader);
                //承载bean定义的容器
                rbd = new RootBeanDefinition(clazz, cargs, pvs);
            }
            else {
                // 承载bean定义的容器
                rbd = new RootBeanDefinition(className, cargs, pvs);
            }

            if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
                String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
                // 记录bean依赖
                rbd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, BEAN_NAME_DELIMITERS, true, true));
            }

            String dependencyCheck = ele.getAttribute(DEPENDENCY_CHECK_ATTRIBUTE);
            if (DEFAULT_VALUE.equals(dependencyCheck)) {
                //若为默认,使用beans标签的默认值
                dependencyCheck = this.defaultDependencyCheck;
            }
            // 将default-dependency-check配置的none | objects | simple | all,转换为内部使用的			 // 数据,默认为none,即不检查
            rbd.setDependencyCheck(getDependencyCheck(dependencyCheck));

            String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
            if (DEFAULT_VALUE.equals(autowire)) {
                  // 若为默认,使用beans标签的默认值
                autowire = this.defaultAutowire;
            }
            // 将default-autowire配置的no | byName | byType | constructor | autodetect,转换             // 为内部使用的数据结果,默认为no,即不自动装配
            rbd.setAutowireMode(getAutowireMode(autowire));

            String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
            if (!initMethodName.equals("")) {
                rbd.setInitMethodName(initMethodName);
            }
            String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
            if (!destroyMethodName.equals("")) {
                rbd.setDestroyMethodName(destroyMethodName);
            }

            bd = rbd;
        }
        else {
            // 如果class为空,实现的bd为ChildBeanDefinition
            bd = new ChildBeanDefinition(parent, pvs);
        }

        if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
            bd.setSingleton(TRUE_VALUE.equals(ele.getAttribute(SINGLETON_ATTRIBUTE)));
        }

        String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
        if (DEFAULT_VALUE.equals(lazyInit) && bd.isSingleton()) {
            // just apply default to singletons, as lazy-init has no meaning for prototypes
            lazyInit = this.defaultLazyInit;
        }
        bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
        bd.setResourceDescription(this.resource.getDescription());
        return bd;
    }
    catch (ClassNotFoundException ex) {
        throw new BeanDefinitionStoreException(this.resource, beanName,"Bean class [" + className + "] not found", ex);
    }
    catch (NoClassDefFoundError err) {
        throw new BeanDefinitionStoreException(this.resource, beanName,"Class that bean class [" + className + "] depends on not found", err);
    }
}
```

loadBeanDefinition、parseBeanDefinition均为protected方法,可用于子类扩展,实际上DefaultXmlBeanDefinitionParser中除了registerBeanDefinitions为public,其余方法均为protected,都可以被重载.

```java
protected MutablePropertyValues getPropertyValueSubElements(String beanName, Element beanEle) {
    NodeList nl = beanEle.getChildNodes();
    // MutablePropertyValues 是PropertyValue的承载类,也是其工具类,通过接口暴露了多种操作
    MutablePropertyValues pvs = new MutablePropertyValues();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element && PROPERTY_ELEMENT.equals(node.getNodeName())) {
            // 处理bean标签的子标签property
            parsePropertyElement(beanName, pvs, (Element) node);
        }
    }
    return pvs;
}
```

```java
protected void parsePropertyElement(String beanName, MutablePropertyValues pvs, Element ele)throws DOMException {
    String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
    if ("".equals(propertyName)) {
        throw new BeanDefinitionStoreException(this.resource, beanName, "Tag 'property' must have a 'name' attribute");
    }
    Object val = getPropertyValue(ele, beanName);
    // PropertyValue 是简单的承载类,虽然如此,并没有使用Map
    pvs.addPropertyValue(new PropertyValue(propertyName, val));
}
```



```java
protected Object getPropertyValue(Element ele, String beanName) {
    // should only have one element child: value, ref, collection
    NodeList nl = ele.getChildNodes();
    Element valueRefOrCollectionElement = null;
    for (int i = 0; i < nl.getLength(); i++) {
        if (nl.item(i) instanceof Element) {
            Element candidateEle = (Element) nl.item(i);
            if (DESCRIPTION_ELEMENT.equals(candidateEle.getTagName())) {
                // keep going: we don't use this value for now
            }
            else {
                // child element is what we're looking for
                valueRefOrCollectionElement = candidateEle;
            }
        }
    }
    if (valueRefOrCollectionElement == null) {
        // 至少得有一个子标签
        throw new BeanDefinitionStoreException(this.resource, beanName,"<property> element must have a subelement like 'value' or 'ref'");
    }
    return parsePropertySubelement(valueRefOrCollectionElement, beanName);
}
```

```java
//返回类型有多种
protected Object parsePropertySubelement(Element ele, String beanName) {
    if (ele.getTagName().equals(BEAN_ELEMENT)) {
        // 递归调用,意味着property里还可以配置bean,且该bean name为 (inner bean definition)
        // createBean中有对其的处理 
        return parseBeanDefinition(ele, "(inner bean definition)");
    }
    else if (ele.getTagName().equals(REF_ELEMENT)) {
        // a generic reference to any name of any bean
        String beanRef = ele.getAttribute(BEAN_REF_ATTRIBUTE);
        if ("".equals(beanRef)) {
            // a reference to the id of another bean in the same XML file
            beanRef = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
            if ("".equals(beanRef)) {
                throw new BeanDefinitionStoreException(this.resource, beanName,"Either 'bean' or 'local' is required for a reference");
            }
        }
        //ref被包装为RuntimeBeanReference,后续会单独做处理
        return new RuntimeBeanReference(beanRef);
    }
    else if (ele.getTagName().equals(IDREF_ELEMENT)) {
        // a generic reference to any name of any bean
        String beanRef = ele.getAttribute(BEAN_REF_ATTRIBUTE);
        if ("".equals(beanRef)) {
            // a reference to the id of another bean in the same XML file
            beanRef = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
            if ("".equals(beanRef)) {
                throw new BeanDefinitionStoreException(this.resource, beanName, "Either 'bean' or 'local' is required for an idref");
            }
        }
        return beanRef;
    }
    else if (ele.getTagName().equals(LIST_ELEMENT)) {
        return getList(ele, beanName);
    }
    else if (ele.getTagName().equals(SET_ELEMENT)) {
        return getSet(ele, beanName);
    }
    else if (ele.getTagName().equals(MAP_ELEMENT)) {
        return getMap(ele, beanName);
    }
    else if (ele.getTagName().equals(PROPS_ELEMENT)) {
        return getProps(ele, beanName);
    }
    else if (ele.getTagName().equals(VALUE_ELEMENT)) {
        // it's a literal value
        return getTextValue(ele, beanName);
    }
    else if (ele.getTagName().equals(NULL_ELEMENT)) {
        // it's a distinguished null value
        // 若需要设置为null需要null标签,而不是空
        return null;
    }
    // 若都不是,终止解析
    throw new BeanDefinitionStoreException(this.resource, beanName, "Unknown subelement of <property>: <" + ele.getTagName() + ">");
}
```

idref和ref用于

```xml
<bean id="" class="">
 <property name="b1" ref="b1" />
</bean>
```

```xml
<bean id="" class="">
 <property name="b1" idref="b1" />
</bean>
```

唯一区别是idref是根据bean的id来绑定的.

< https://stackoverflow.com/questions/1767831/ref-vs-idref-attributes-in-spring-bean-declaration >

getList、getSet、getMap返回的是ManagedList、ManagedSet、ManagedMap,1.0中只用来做标记,并未添加更多特性,不过留下了扩展点.spring有大量类似这种的处理逻辑,例如PropertyValue、MutablePropertyValues等,通过增加抽象层,来提供扩展性;通过接口描述能力等等,值得学习.

再来学习构造器子节点的解析.

```java
protected ConstructorArgumentValues getConstructorArgSubElements(String beanName, Element beanEle)throws ClassNotFoundException {
    NodeList nl = beanEle.getChildNodes();
    ConstructorArgumentValues cargs = new ConstructorArgumentValues();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element && CONSTRUCTOR_ARG_ELEMENT.equals(node.getNodeName())) {
            parseConstructorArgElement(beanName, cargs, (Element) node);
        }
    }
    return cargs;
}
```

ConstructorArgumentValues为构造器方法的承载类,存储了按index,class的构造方法参数索引、类型,通过indexedArgumentValues、genericArgumentValues存储,而具体的值及其类型通过其内部类ValueHolder记录.

```java
protected void parseConstructorArgElement(String beanName, ConstructorArgumentValues cargs, Element ele)throws DOMException, ClassNotFoundException {
    Object val = getPropertyValue(ele, beanName);
    String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
    String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
    if (!"".equals(indexAttr)) {
        try {
            int index = Integer.parseInt(indexAttr);
            if (index < 0) {
                throw new BeanDefinitionStoreException(this.resource, beanName, "'index' cannot be lower than 0");
            }
            if (!"".equals(typeAttr)) {
                cargs.addIndexedArgumentValue(index, val, typeAttr);
            }
            else {
                cargs.addIndexedArgumentValue(index, val);
            }
        }
        catch (NumberFormatException ex) {
            throw new BeanDefinitionStoreException(this.resource, beanName,"Attribute 'index' of tag 'constructor-arg' must be an integer");
        }
    }
    else {
        if (!"".equals(typeAttr)) {
            cargs.addGenericArgumentValue(val, typeAttr);
        }
        else {
            cargs.addGenericArgumentValue(val);
        }
    }
}
```

RootBeanDefinition是bean在spring容器中的表达.不论是MutablePropertyValues还是ConstructorArgumentValues都会转化到其中.bean的autowireMode、dependencyCheck默认值也是在这里表达.通过继承AbstractBeanDefinition实现,这是模板方法的直接体现.接着DefaultXmlBeanDefinitionParser的parseBeanDefinition方法

```java
if (this.beanClassLoader != null) {
    Class clazz = Class.forName(className, true, this.beanClassLoader);
    rbd = new RootBeanDefinition(clazz, cargs, pvs);
}else {
    rbd = new RootBeanDefinition(className, cargs, pvs);
}
```

```java
public RootBeanDefinition(Class beanClass, ConstructorArgumentValues cargs, MutablePropertyValues pvs) {
    //pvs存储在AbstractBeanDefinition中,若为空,new一个空对象
    super(pvs);
    //beanClass类型为Object,可以接受Class String
    this.beanClass = beanClass;
    this.constructorArgumentValues = cargs;
}
```

DefaultXmlBeanDefinitionParser的loadBeanDefinition中

```java
this.beanFactory.registerBeanDefinition(id, beanDefinition);
for (Iterator it = aliases.iterator(); it.hasNext();) {
    this.beanFactory.registerAlias(id, (String) it.next());
}
```

this.beanFactory类型为BeanDefinitionRegistry,而XmlBeanFactory是其实现类,也就是将前述流程中beanDefinition注册到factory中.后续的使用又通过getBeanDefinition使用,这种设计方法值得借鉴.

```java
public interface BeanDefinitionRegistry {
	int getBeanDefinitionCount();

	String[] getBeanDefinitionNames();

	boolean containsBeanDefinition(String name);

	/**
	 * Return the BeanDefinition for the given bean name.
	 * @param name name of the bean to find a definition for
	 * @return the BeanDefinition for the given name (never null)
	 * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
	 * if the bean definition cannot be resolved
	 * @throws BeansException in case of errors
	 */
	BeanDefinition getBeanDefinition(String name) throws BeansException;

	/**
	 * Register a new bean definition with this registry.
	 * Must support RootBeanDefinition and ChildBeanDefinition.
	 * @param name the name of the bean instance to register
	 * @param beanDefinition definition of the bean instance to register
	 * @throws BeansException if the bean definition is invalid
	 * @see RootBeanDefinition
	 * @see ChildBeanDefinition
	 */
	void registerBeanDefinition(String name, BeanDefinition beanDefinition)
			throws BeansException;
    
	String[] getAliases(String name) throws NoSuchBeanDefinitionException;

	void registerAlias(String name, String alias) throws BeansException;
}
```

其具体实现在XmlBeanFactory继承结构中DefaultListableBeanFactory层,如下:

```java
public void registerBeanDefinition(String name, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {
    if (beanDefinition instanceof AbstractBeanDefinition) {
        // 前述中只有RootBeanDefinition ChildBeanDefinition,也就是都要validate
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), name, "Validation of bean definition with name failed", ex);
        }
    }
    // beanDefinitionMap就是一个简单的map
    Object oldBeanDefinition = this.beanDefinitionMap.get(name);
    if (oldBeanDefinition != null) {
        // 是否允许重载
        if (!this.allowBeanDefinitionOverriding) {
            throw new BeanDefinitionStoreException("Cannot register bean definition [" + beanDefinition + "] for bean '" +name + "': there's already [" + oldBeanDefinition + "] bound");
        }
        else {
           // log
        }
    }
    else {
        this.beanDefinitionNames.add(name);
    }
    this.beanDefinitionMap.put(name, beanDefinition);
}
```

registerAlias与registerBeanDefinition都是BeanDefinitionRegistry接口中的方法，但是其默认实现在AbstractBeanFactory,该Factory承载了factory的基础方法.

至此xml文件中bean相关配置已转移至beanFactory中,以下是对其的使用.

```java
Object bean = factory.getBean("exampleBean", ExampleBean.class);
```

getBean是容器的核心方法,其实现在AbstractBeanFactory

```java
public Object getBean(String name, Class requiredType) throws BeansException {
    Object bean = this.getBean(name);
    if (!requiredType.isAssignableFrom(bean.getClass())) {
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean);
    } else {
        return bean;
}
```

```java
public Object getBean(String name) throws BeansException {
    // 主要是去除factoryBean的&,以及获取bean在容器中的名字
    String beanName = transformedBeanName(name);
    // eagerly check singleton cache for manually registered singletons
    Object sharedInstance = this.singletonCache.get(beanName);
    if (sharedInstance != null) {
        return getObjectForSharedInstance(name, sharedInstance);
    }
    else {
        // check if bean definition exists
        RootBeanDefinition mergedBeanDefinition = null;
        try {
            mergedBeanDefinition = getMergedBeanDefinition(beanName, false);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // not found -> check parent
            if (this.parentBeanFactory != null) {
                return this.parentBeanFactory.getBean(name);
            }
            throw ex;
        }
        // create bean instance
        if (mergedBeanDefinition.isSingleton()) {
            // 单例
            // 双重锁
            synchronized (this.singletonCache) {
                // re-check singleton cache within synchronized block
                sharedInstance = this.singletonCache.get(beanName);
                if (sharedInstance == null) {
                    sharedInstance = createBean(beanName, mergedBeanDefinition);
                    addSingleton(beanName, sharedInstance);
                }
            }
            return getObjectForSharedInstance(name, sharedInstance);
        }
        else {
            // 非单例
            return createBean(name, mergedBeanDefinition);
        }
    }
}
```



```java
protected Object getObjectForSharedInstance(String name, Object beanInstance) {
    String beanName = transformedBeanName(name);

    // Don't let calling code try to dereference the
    // bean factory if the bean isn't a factory
    if (isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
        throw new BeanIsNotAFactoryException(beanName, beanInstance);
    }

    // Now we have the bean instance, which may be a normal bean
    // or a FactoryBean. If it's a FactoryBean, we use it to
    // create a bean instance, unless the caller actually wants
    // a reference to the factory.
    if (beanInstance instanceof FactoryBean) {
        if (!isFactoryDereference(name)) {
            //如果bean是factoryBean,而想要获取的不是factoryBean本身,就使用其getObject方法生产bean
            // return bean instance from factory
            FactoryBean factory = (FactoryBean) beanInstance;
            try {
                beanInstance = factory.getObject();
            }
            catch (BeansException ex) {
                throw ex;
            }
            catch (Exception ex) {
                throw new BeanCreationException("FactoryBean threw exception on object creation", ex);
            }
            if (beanInstance == null) {
                throw new FactoryBeanCircularReferenceException("Factory bean '" + beanName + "' returned null object - " +"possible cause: not fully initialized due to circular bean reference");
            }
        }
        else {
            // the user wants the factory itself
        }
    }
    return beanInstance;
}
```

```java
// 此时includingAncestors为false
public RootBeanDefinition getMergedBeanDefinition(String beanName, boolean includingAncestors) throws BeansException {
    try {
        // getBeanDefinition只是简单的去当前容器beanDefinitionMap中获取beanDefinition
        // 且如果没有会抛出NoSuchBeanDefinitionException
        // getMergedBeanDefinition 主要为应对有父类容器的beanDefinition的合并
        return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
    }
    catch (NoSuchBeanDefinitionException ex) {
        // 在有父类容器的场景下生效,实质就是去父类容器中找definition
        // 为什么要求是AbstractAutowireCapableBeanFactory?
        if (includingAncestors && getParentBeanFactory() instanceof AbstractAutowireCapableBeanFactory) {
            return ((AbstractAutowireCapableBeanFactory) getParentBeanFactory()).getMergedBeanDefinition(beanName, true);
        }
        else {
            throw ex;
        }
    }
}
```



```java
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd) {
    if (bd instanceof RootBeanDefinition) {
        // 如果是RootBeanDefinition 说明没有父类容器,也就不存在merge
        return (RootBeanDefinition) bd;
    }
    else if (bd instanceof ChildBeanDefinition) {
        // 参考DefaultXmlBeanDefinitionParser中parseBeanDefinition方法
        ChildBeanDefinition cbd = (ChildBeanDefinition) bd;
        // deep copy
        // 递归查找父类容器
        RootBeanDefinition rbd = new RootBeanDefinition(getMergedBeanDefinition(cbd.getParentName(), true));
        // override properties
        for (int i = 0; i < cbd.getPropertyValues().getPropertyValues().length; i++) {
            //有同名的就更新,否则直接添加
            rbd.getPropertyValues().addPropertyValue(cbd.getPropertyValues().getPropertyValues()[i]);
        }
        // setDependsOn setDependencyCheck setAutowireMode setInitMethodName setDestroyMethodName 未被重载
        // override settings
        rbd.setSingleton(cbd.isSingleton());
        rbd.setLazyInit(cbd.isLazyInit());
        rbd.setResourceDescription(cbd.getResourceDescription());
        return rbd;
    }
    else {
        throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
             "Definition is neither a RootBeanDefinition nor a ChildBeanDefinition");
    }
}
```

```java
public RootBeanDefinition(RootBeanDefinition other) {
    super(new MutablePropertyValues(other.getPropertyValues()));
    this.beanClass = other.beanClass;
    this.constructorArgumentValues = other.constructorArgumentValues;
    // 使用set方法,而不是直接赋值,可以有更多的逻辑控制
    setSingleton(other.isSingleton());
    setLazyInit(other.isLazyInit());
    setDependsOn(other.getDependsOn());
    setDependencyCheck(other.getDependencyCheck());
    setAutowireMode(other.getAutowireMode());
    setInitMethodName(other.getInitMethodName());
    setDestroyMethodName(other.getDestroyMethodName());
}
```

继续查看createBean过程,位于AbstractBeanFactory

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mergedBeanDefinition) throws BeansException;
```

默认实现在AbstractAutowireCapableBeanFactory中,为protected方法

```java
protected Object createBean(String beanName, RootBeanDefinition mergedBeanDefinition) throws BeansException {
    if (mergedBeanDefinition.getDependsOn() != null) {
        for (int i = 0; i < mergedBeanDefinition.getDependsOn().length; i++) {
            // 优先创建依赖项
            // guarantee initialization of beans that the current one depends on
            getBean(mergedBeanDefinition.getDependsOn()[i]);
        }
    }

    BeanWrapper instanceWrapper = null;
    // 构造器注入
    if (mergedBeanDefinition.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
        mergedBeanDefinition.hasConstructorArgumentValues()) {
        // 1. autowireConstructor
        instanceWrapper = autowireConstructor(beanName, mergedBeanDefinition);
    }
    else {
        // 2. BeanWrapperImpl
        instanceWrapper = new BeanWrapperImpl(mergedBeanDefinition.getBeanClass());
        initBeanWrapper(instanceWrapper);
    }
    Object bean = instanceWrapper.getWrappedInstance();

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    if (mergedBeanDefinition.isSingleton()) {
        addSingleton(beanName, bean);
    }

    // 3. populateBean
    // 填充bean 属性
    populateBean(beanName, mergedBeanDefinition, instanceWrapper);

    try {
        if (bean instanceof BeanNameAware) {
            // 实现BeanNameAware接口就能获取bean name的逻辑
            ((BeanNameAware) bean).setBeanName(beanName);
        }

        if (bean instanceof BeanFactoryAware) {
             // 实现BeanFactoryAware接口就能获取bean factory的逻辑
            ((BeanFactoryAware) bean).setBeanFactory(this);
        }

        // 前置处理机,重要扩展点
        bean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        // 4. invokeInitMethods
        // 初始化方法
        invokeInitMethods(bean, beanName, mergedBeanDefinition);
        // 后置处理机,重要扩展点
        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
    }
    catch (InvocationTargetException ex) {
        throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName, "Initialization of bean failed", ex.getTargetException());
    }
    catch (Exception ex) {
        throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    }
    return bean;
}
```

1.autowireConstructor

```java
protected BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mergedBeanDefinition)
			throws BeansException {
    ConstructorArgumentValues cargs=mergedBeanDefinition.getConstructorArgumentValues();
    // 真正使用的数据多数使用resolved开头
    ConstructorArgumentValues resolvedValues = new ConstructorArgumentValues();

    int minNrOfArgs = 0;
    if (cargs != null) {
        // 参数总数量
        minNrOfArgs = cargs.getNrOfArguments();
        for (Iterator it = cargs.getIndexedArgumentValues().entrySet().iterator(); it.hasNext();) {
            Map.Entry entry = (Map.Entry) it.next();
            int index = ((Integer) entry.getKey()).intValue();
            if (index < 0) {
                throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName, "Invalid constructor argument index: " + index);
            }
            if (index > minNrOfArgs) {
                // TODO
                minNrOfArgs = index + 1;
            }
            String argName = "constructor argument with index " + index;
            ConstructorArgumentValues.ValueHolder valueHolder = (ConstructorArgumentValues.ValueHolder) entry.getValue();
            // **IfNecessary 也是spring惯用处理逻辑 
            Object resolvedValue = resolveValueIfNecessary(beanName, mergedBeanDefinition, argName, valueHolder.getValue());
            resolvedValues.addIndexedArgumentValue(index, resolvedValue, valueHolder.getType());
        }
        for (Iterator it = cargs.getGenericArgumentValues().iterator(); it.hasNext();) {
            ConstructorArgumentValues.ValueHolder valueHolder = (ConstructorArgumentValues.ValueHolder) it.next();
            String argName = "constructor argument";
            Object resolvedValue = resolveValueIfNecessary(beanName, mergedBeanDefinition, argName, valueHolder.getValue());
            resolvedValues.addGenericArgumentValue(resolvedValue, valueHolder.getType());
        }
    }

    Constructor[] constructors = mergedBeanDefinition.getBeanClass().getConstructors();
    // 构造方法参数数量越多优先级越高
    Arrays.sort(constructors, new Comparator() {
        public int compare(Object o1, Object o2) {
            int c1pl = ((Constructor) o1).getParameterTypes().length;
            int c2pl = ((Constructor) o2).getParameterTypes().length;
            return (new Integer(c1pl)).compareTo(new Integer(c2pl)) * -1;
        }
    });

    BeanWrapperImpl bw = new BeanWrapperImpl();
    // 向bw中注册自定义属性编辑器(实质就是将xml中的字符串通过自定义逻辑转换为bean的属性)
    // spring1.0 内置了ClassEditor FileEditor LocalEditor PropertiesEditor StringArrayPropertyEditor URLEditor,通过内省实现,可参考
    initBeanWrapper(bw);
    Constructor constructorToUse = null;
    Object[] argsToUse = null;
    int minTypeDiffWeight = Integer.MAX_VALUE;
    for (int i = 0; i < constructors.length; i++) {
        try {
            Constructor constructor = constructors[i];
            if (constructor.getParameterTypes().length < minNrOfArgs) {
                throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName, minNrOfArgs + " constructor arguments specified but no matching constructor found in bean '" +beanName + "' (hint: specify index arguments for simple parameters to avoid type ambiguities)");
            }
            Class[] argTypes = constructor.getParameterTypes();
            Object[] args = new Object[argTypes.length];
            for (int j = 0; j < argTypes.length; j++) {
                // 若index不满足,会使用GenericArgumentValue
                ConstructorArgumentValues.ValueHolder valueHolder = resolvedValues.getArgumentValue(j, argTypes[j]);
                if (valueHolder != null) {
                    // synchronize if custom editors are registered
                    // necessary because PropertyEditors are not thread-safe
                    if (!getCustomEditors().isEmpty()) {
                        synchronized (this) {
                            args[j] = bw.doTypeConversionIfNecessary(valueHolder.getValue(), argTypes[j]);
                        }
                    }
                    else {
                        // 见后续分析
                        args[j] = bw.doTypeConversionIfNecessary(valueHolder.getValue(), argTypes[j]);
                    }
                }
                else {
                    if (mergedBeanDefinition.getResolvedAutowireMode() != RootBeanDefinition.AUTOWIRE_CONSTRUCTOR) {
                        throw new UnsatisfiedDependencyException(beanName, j, argTypes[j],"Did you specify the correct bean references as generic constructor arguments?");
                    }
                    // 在当前容器中寻找指定class的bean,若有父类容器,也需一并返回
                    Map matchingBeans = findMatchingBeans(argTypes[j]);
                    // 只能有且仅一个候选
                    if (matchingBeans == null || matchingBeans.size() != 1) {
                        throw new UnsatisfiedDependencyException(beanName, j, argTypes[j], "There are " + matchingBeans.size() + " beans of type [" + argTypes[j] + "] for autowiring constructor. " + "There should have been 1 to be able to autowire constructor of bean '" + beanName + "'.");
                    }
                    args[j] = matchingBeans.values().iterator().next();
                }
            }
            // 计算构造方法参数权重
            // 如果值的类型不是构造方法中参数类型的子类,直接返回 Integer.MAX_VALUE.如果构造方法类型和实际值类型是否相同，或是否是其超类或超接口,或者构造方法中类型的父类型和实际值类型有同样关系,如果是则权重+1,直至不匹配
            int typeDiffWeight = getTypeDifferenceWeight(argTypes, args);
            if (typeDiffWeight < minTypeDiffWeight) {
                // typeDiffWeight越小,构造方法的实际值越特殊
                constructorToUse = constructor;
                argsToUse = args;
                minTypeDiffWeight = typeDiffWeight;
            }
        }
        catch (BeansException ex) {
            if (i == constructors.length - 1 && constructorToUse == null) {
                // all constructors tried
                throw ex;
            }
            else {
                // swallow and try next constructor
            }
        }
    }

    if (constructorToUse == null) {
        throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName,"Could not resolve matching constructor");
    }
    bw.setWrappedInstance(BeanUtils.instantiateClass(constructorToUse, argsToUse));
    return bw;
}

/**
	 * Given a PropertyValue, return a value, resolving any references to other
	 * beans in the factory if necessary. The value could be:
	 * <li>A BeanDefinition, which leads to the creation of a corresponding
	 * new bean instance. Singleton flags and names of such "inner beans"
	 * are always ignored: Inner beans are anonymous prototypes.
	 * <li>A RuntimeBeanReference, which must be resolved.
	 * <li>A ManagedList. This is a special collection that may contain
	 * RuntimeBeanReferences or Collections that will need to be resolved.
	 * <li>A ManagedSet. May also contain RuntimeBeanReferences or
	 * Collections that will need to be resolved.
	 * <li>A ManagedMap. In this case the value may be a RuntimeBeanReference
	 * or Collection that will need to be resolved.
	 * <li>An ordinary object or null, in which case it's left alone.
	 */
protected Object resolveValueIfNecessary(String beanName, RootBeanDefinition mergedBeanDefinition,String argName, Object value) throws BeansException {
    // We must check each PropertyValue to see whether it
    // requires a runtime reference to another bean to be resolved.
    // If it does, we'll attempt to instantiate the bean and set the reference.
    // bean 属性,bean name为 (inner bean definition)
    if (value instanceof AbstractBeanDefinition) {
        BeanDefinition bd = (BeanDefinition) value;
        if (bd instanceof AbstractBeanDefinition) {
            // an inner bean should never be cached as singleton
            ((AbstractBeanDefinition) bd).setSingleton(false);
        }
        String innerBeanName = "(inner bean for property '" + beanName + "." + argName + "')";
        Object bean = createBean(innerBeanName, getMergedBeanDefinition(innerBeanName, bd));
        if (bean instanceof DisposableBean) {
            // keep reference to inner bean, to be able to destroy it on factory shutdown
            this.disposableInnerBeans.add(bean);
        }
        return getObjectForSharedInstance(innerBeanName, bean);
    }
    // ref 属性
    else if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        // getBean ref的bean
        return resolveReference(mergedBeanDefinition, beanName, argName, ref);
    }
    // list 属性
    else if (value instanceof ManagedList) {
        // Convert from managed list. This is a special container that may
        // contain runtime bean references. May need to resolve references.
        return resolveManagedList(beanName, mergedBeanDefinition, argName, (ManagedList) value);
    }
    // set 属性
    else if (value instanceof ManagedSet) {
        // Convert from managed set. This is a special container that may
        // contain runtime bean references. May need to resolve references.
        return resolveManagedSet(beanName, mergedBeanDefinition, argName, (ManagedSet) value);
    }
    // map 属性
    else if (value instanceof ManagedMap) {
        // Convert from managed map. This is a special container that may
        // contain runtime bean references. May need to resolve references.
        ManagedMap mm = (ManagedMap) value;
        return resolveManagedMap(beanName, mergedBeanDefinition, argName, mm);
    }
    else {
        // idref props value null 属性
        // no need to resolve value
        return value;
    }
}

// resolveManagedSet resolveManagedMap类似
protected List resolveManagedList(String beanName, RootBeanDefinition mergedBeanDefinition,String argName, ManagedList ml) throws BeansException {
    // 返回的是ArrayList,精妙设计
    List resolved = new ArrayList();
    for (int i = 0; i < ml.size(); i++) {
        // 还可嵌套
        resolved.add(resolveValueIfNecessary(beanName, mergedBeanDefinition, argName + "[" + i + "]", ml.get(i)));
    }
    return resolved;
}
```

自定义属性编辑器可参考:[Spring PropertyEditor分析](https://blog.csdn.net/pentiumchen/article/details/44026575) [Spring核心——字符串到实体转换](https://www.chkui.com/article/spring/spring_core_string_to_entity)

2.BeanWrapperImpl

BeanWrapperImpl类是对BeanWrapper接口的默认实现，它承载了一个bean对象，缓存了bean的内省结果，并可以访问bean的属性、设置bean的属性值(通过内省)。BeanWrapperImpl类提供了许多默认属性编辑器，支持多种不同类型的类型转换，可以将数组、集合类型的属性转换成指定特殊类型的数组或集合。用户也可以注册自定义的属性编辑器在BeanWrapperImpl中。

```java
public BeanWrapperImpl(Class clazz) throws BeansException {
    setWrappedInstance(BeanUtils.instantiateClass(clazz));
}

public void setWrappedInstance(Object object) throws BeansException {
    if (object == null) {
        throw new FatalBeanException("Cannot set BeanWrapperImpl target to a null object");
    }
    this.object = object;
    this.nestedBeanWrappers = null;
    if (this.cachedIntrospectionResults == null ||
        !this.cachedIntrospectionResults.getBeanClass().equals(object.getClass())) {
        // 缓存内省结果,避免重复开销,spring中随处可见的缓存实现,值得借鉴
        this.cachedIntrospectionResults = CachedIntrospectionResults.forClass(object.getClass());
    }
}

// 目的很简单,将newValue转换成要求的type
public Object doTypeConversionIfNecessary(Object newValue, Class requiredType) throws BeansException {
    return doTypeConversionIfNecessary(null, null, null, newValue, requiredType);
}

protected Object doTypeConversionIfNecessary(String propertyName, String propertyDescriptor, Object oldValue, Object newValue, Class requiredType) throws BeansException {
    if (newValue != null) {
        if (requiredType.isArray()) {
            // 数组class对象的getComponentType方法可以取得一个数组的Class对象,通过 Array.newInstance()可反射生成数组对象,若不是数组class,返回的是null
            // convert individual elements to array elements
            Class componentType = requiredType.getComponentType();
            if (newValue instanceof List) {
                List list = (List) newValue;
                Object result = Array.newInstance(componentType, list.size());
                for (int i = 0; i < list.size(); i++) {
                    // 复用转换逻辑
                    Object value = doTypeConversionIfNecessary(propertyName, propertyName + "[" + i + "]",null, list.get(i), componentType);
                    Array.set(result, i, value);
                }
                return result;
            }
            else if (newValue instanceof Object[]) {
                Object[] array = (Object[]) newValue;
                Object result = Array.newInstance(componentType, array.length);
                for (int i = 0; i < array.length; i++) {
                    Object value = doTypeConversionIfNecessary(propertyName, propertyName + "[" + i + "]", null, array[i], componentType);
                    Array.set(result, i, value);
                }
                return result;
            }
        }

        // custom editor for this type?
        PropertyEditor pe = findCustomEditor(requiredType, propertyName);

        // value not of required type?
        if (pe != null || !requiredType.isAssignableFrom(newValue.getClass())) {
            // 如果类型不匹配或customEditor不空,就是necessary
            if (newValue instanceof String[]) {
                newValue = StringUtils.arrayToCommaDelimitedString((String[]) newValue);
            }

            if (newValue instanceof String) {
                if (pe == null) {
                    // no custom editor -> check BeanWrapper's default editors
                    // 1.0中默认只有String->Class File Locale Properties String[] URL
                    pe = findDefaultEditor(requiredType);
                    if (pe == null) {
                      // no BeanWrapper default editor -> check standard JavaBean editors
                      // 寻找jdk中内置的editor,只有Byte Short Integer Long Boolean Float Double
                        pe = PropertyEditorManager.findEditor(requiredType);
                    }
                }
                if (pe != null) {
                    // use PropertyEditor's setAsText in case of a String value
                    try {
                        pe.setAsText((String) newValue);
                        newValue = pe.getValue();
                    }
                    catch (IllegalArgumentException ex) {
                        throw new TypeMismatchException(createPropertyChangeEvent(propertyDescriptor, oldValue, newValue),
                                                        requiredType, ex);
                    }
                }
                else {
                    throw new TypeMismatchException(createPropertyChangeEvent(propertyDescriptor, oldValue, newValue),
                                                    requiredType);
                }
            }

            else if (pe != null) {
                // Not a String -> use PropertyEditor's setValue.
                // With standard PropertyEditors, this will return the very same object;
                // we just want to allow special PropertyEditors to override setValue
                // for type conversion from non-String values to the required type.
                try {
                    pe.setValue(newValue);
                    newValue = pe.getValue();
                }
                catch (IllegalArgumentException ex) {
                    throw new TypeMismatchException(createPropertyChangeEvent(propertyDescriptor, oldValue, newValue),
                                                    requiredType, ex);
                }
            }
        }

        if (requiredType.isArray() && !newValue.getClass().isArray()) {
            Class componentType = requiredType.getComponentType();
            Object result = Array.newInstance(componentType, 1) ;
            Object val = doTypeConversionIfNecessary(propertyName, propertyName + "[" + 0 + "]",null, newValue, componentType);
            Array.set(result, 0, val) ;
            return result;
        }
    }
    return newValue;
}


public PropertyEditor findCustomEditor(Class requiredType, String propertyPath) {
    if (propertyPath != null) {
        BeanWrapperImpl bw = getBeanWrapperForPropertyPath(propertyPath);
        return bw.doFindCustomEditor(requiredType, getFinalPath(propertyPath));
    }
    else {
        return doFindCustomEditor(requiredType, propertyPath);
    }
}

// 返回的是最终的BeanWrapperImpl
private BeanWrapperImpl getBeanWrapperForPropertyPath(String propertyPath) {
    int pos = propertyPath.indexOf(NESTED_PROPERTY_SEPARATOR);
    // Handle nested properties recursively
    if (pos > -1) {
        // 如果propertyPath为managingDirector.salary,nestedProperty为managingDirector,nestedPath为salary
        String nestedProperty = propertyPath.substring(0, pos);
        String nestedPath = propertyPath.substring(pos + 1);
        // 如果propertyPath为managingDirector.salary,nestedBw就是managingDirector的bw
        BeanWrapperImpl nestedBw = getNestedBeanWrapper(nestedProperty);
        return nestedBw.getBeanWrapperForPropertyPath(nestedPath);
    }
    else {
        return this;
    }
}

private BeanWrapperImpl getNestedBeanWrapper(String nestedProperty) {
    if (this.nestedBeanWrappers == null) {
        this.nestedBeanWrappers = new HashMap();
    }
    // get value of bean property
    String[] tokens = getPropertyNameTokens(nestedProperty);
    Object propertyValue = getPropertyValue(tokens[0], tokens[1], tokens[2]);
    String canonicalName = tokens[0];
    if (propertyValue == null) {
        throw new NullValueInNestedPathException(getWrappedClass(), canonicalName);
    }

    // lookup cached sub-BeanWrapper, create new one if not found
    // 随处可见的缓存
    BeanWrapperImpl nestedBw = (BeanWrapperImpl) this.nestedBeanWrappers.get(canonicalName);
    if (nestedBw == null) {
        nestedBw = new BeanWrapperImpl(propertyValue, this.nestedPath + canonicalName + NESTED_PROPERTY_SEPARATOR);
        // inherit all type-specific PropertyEditors
        if (this.customEditors != null) {
            for (Iterator it = this.customEditors.keySet().iterator(); it.hasNext();) {
                Object key = it.next();
                if (key instanceof Class) {
                    Class requiredType = (Class) key;
                    PropertyEditor propertyEditor = (PropertyEditor) this.customEditors.get(key);
                    // customEditor配置一次,会在所有BeanWrapperImpl中生效
                    nestedBw.registerCustomEditor(requiredType, null, propertyEditor);
                }
            }
        }
        this.nestedBeanWrappers.put(canonicalName, nestedBw);
    }
    else {
      // log
    }
    return nestedBw;
}

private String[] getPropertyNameTokens(String propertyName) {
    String actualName = propertyName;
    // 实际上就是索引,适用于collection
    String key = null;
    int keyStart = propertyName.indexOf('[');
    if (keyStart != -1 && propertyName.endsWith("]")) {
        actualName = propertyName.substring(0, keyStart);
        key = propertyName.substring(keyStart + 1, propertyName.length() - 1);
        if (key.startsWith("'") && key.endsWith("'")) {
            key = key.substring(1, key.length() - 1);
        }
        else if (key.startsWith("\"") && key.endsWith("\"")) {
            key = key.substring(1, key.length() - 1);
        }
    }
    String canonicalName = actualName;
    if (key != null) {
        canonicalName += "[" + key + "]";
    }
    return new String[] {canonicalName, actualName, key};
}

private PropertyEditor doFindCustomEditor(Class requiredType, String propertyName) {
    if (this.customEditors == null) {
        return null;
    }
    if (propertyName != null) {
        // check property-specific editor first
        PropertyDescriptor descriptor = null;
        try {
            descriptor = getPropertyDescriptor(propertyName);
            // 每个customEditor都是PropertyEditor
            PropertyEditor editor = (PropertyEditor)this.customEditors.get(propertyName);
            if (editor != null) {
                // consistency check
                if (requiredType != null) {
                    if (!descriptor.getPropertyType().isAssignableFrom(requiredType)) {
                        throw new IllegalArgumentException("Types do not match: required=" + requiredType.getName() +", found=" + descriptor.getPropertyType());
                    }
                }
                return editor;
            }
            else {
                if (requiredType == null) {
                    // try property type
                    requiredType = descriptor.getPropertyType();
                }
            }
        }
        catch (BeansException ex) {
            // probably an indexed or mapped property
            // we need to retrieve the value to determine the type
            requiredType = getPropertyValue(propertyName).getClass();
        }
    }
    // no property-specific editor -> check type-specific editor
    return (PropertyEditor) this.customEditors.get(requiredType);
}
```

BeanWrapperImpl举例

````java
public class Company {
    private String name;
    private Employee managingDirector;
    // 省略get set
}
public class Employee {
    private float salary;
    // 省略get set
}
public class Person {
    private int age;
    private String name;
    private List<String> friends;
 	// 省略get set
}

public static void main(String[] args) {
    Company c = new Company();
    BeanWrapper bwComp = new BeanWrapperImpl(c);
    // setting the company name...
    bwComp.setPropertyValue("name", "Some Company Inc.");
    // ... can also be done like this:
    PropertyValue v = new PropertyValue("name", "Some Company Inc.");
    bwComp.setPropertyValue(v);
    // ok, let's create the director and tie it to the company:
    Employee jim = new Employee();
    BeanWrapper bwJim = new BeanWrapperImpl(jim);
    Person person = new Person();
    BeanWrapper bwPerson = new BeanWrapperImpl(person);
    bwPerson.setPropertyValue("name", "Jim Stravinsky");
    List<String> friends = new ArrayList<>();
    friends.add("Steve");
    friends.add("Jobs");
    bwPerson.setPropertyValue("friends", friends);
    bwJim.setPropertyValue("person", person);
    bwComp.setPropertyValue("managingDirector", jim);
    // retrieving the salary of the managingDirector through the company
    Float salary = (Float) bwComp.getPropertyValue("managingDirector.salary");
    String name = (String) bwComp.getPropertyValue("managingDirector.person.name");
    String friends0 = (String) bwComp.getPropertyValue("managingDirector.person.friends[0]");
    String friends1 = (String) bwComp.getPropertyValue("managingDirector.person.friends['1']");
    System.out.println(friends0 + " " + friends1);
}
````

3.populateBean bean属性填充

```java
protected void populateBean(String beanName, RootBeanDefinition mergedBeanDefinition, BeanWrapper bw) {
    PropertyValues pvs = mergedBeanDefinition.getPropertyValues();

    if (mergedBeanDefinition.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
        mergedBeanDefinition.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        // 深度复制
        MutablePropertyValues mpvs = new MutablePropertyValues(pvs);

        // add property values based on autowire by name if it's applied
        if (mergedBeanDefinition.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mergedBeanDefinition, bw, mpvs);
        }

        // add property values based on autowire by type if it's applied
        if (mergedBeanDefinition.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mergedBeanDefinition, bw, mpvs);
        }

        // 整体替换,值得借鉴
        pvs = mpvs;
    }

    dependencyCheck(beanName, mergedBeanDefinition, bw, pvs);
    applyPropertyValues(beanName, mergedBeanDefinition, bw, pvs);
}

protected void autowireByName(String beanName, RootBeanDefinition mergedBeanDefinition,BeanWrapper bw, MutablePropertyValues pvs) {
    // 获取不为简单类型 不是被忽略的类型 且value为null
    String[] propertyNames = unsatisfiedObjectProperties(mergedBeanDefinition, bw);
    for (int i = 0; i < propertyNames.length; i++) {
        String propertyName = propertyNames[i];
        if (containsBean(propertyName)) {
            Object bean = getBean(propertyName);
            // 实际注入还未发生
            pvs.addPropertyValue(propertyName, bean);
        }
        else {
            //
        }
    }
}

protected void autowireByType(String beanName, RootBeanDefinition mergedBeanDefinition,
BeanWrapper bw, MutablePropertyValues pvs) {
    String[] propertyNames = unsatisfiedObjectProperties(mergedBeanDefinition, bw);
    for (int i = 0; i < propertyNames.length; i++) {
        String propertyName = propertyNames[i];
        // look for a matching type
        Class requiredType = bw.getPropertyDescriptor(propertyName).getPropertyType();
        // 在构造器注入中已有使用,在容器上下文中寻找
        Map matchingBeans = findMatchingBeans(requiredType);
        // 有且只能有一个候选
        if (matchingBeans != null && matchingBeans.size() == 1) {
            // 实际注入还未发生
            pvs.addPropertyValue(propertyName, matchingBeans.values().iterator().next());
        }
        else if (matchingBeans != null && matchingBeans.size() > 1) {
            throw new UnsatisfiedDependencyException(beanName, propertyName, "There are " + matchingBeans.size() + " beans of type [" + requiredType + "] for autowire by type. " + "There should have been 1 to be able to autowire property '" + propertyName + "' of bean '" + beanName + "'.");
        }
        else {
           //
        }
    }
}

/**
* Return an array of object-type property names that are unsatisfied.
* These are probably unsatisfied references to other beans in the
* factory. Does not include simple properties like primitives or Strings.
*/
protected String[] unsatisfiedObjectProperties(RootBeanDefinition mergedBeanDefinition, BeanWrapper bw) {
    Set result = new TreeSet();
    Set ignoreTypes = getIgnoredDependencyTypes();
    PropertyDescriptor[] pds = bw.getPropertyDescriptors();
    for (int i = 0; i < pds.length; i++) {
        String name = pds[i].getName();
        if (pds[i].getWriteMethod() != null &&
            !BeanUtils.isSimpleProperty(pds[i].getPropertyType()) &&
            !ignoreTypes.contains(pds[i].getPropertyType()) &&
            mergedBeanDefinition.getPropertyValues().getPropertyValue(name) == null) {
            result.add(name);
        }
    }
    return (String[]) result.toArray(new String[result.size()]);
}

protected void applyPropertyValues(String beanName, RootBeanDefinition mergedBeanDefinition, BeanWrapper bw, PropertyValues pvs) throws BeansException {
    if (pvs == null) {
        return;
    }
    MutablePropertyValues deepCopy = new MutablePropertyValues(pvs);
    PropertyValue[] pvals = deepCopy.getPropertyValues();
    for (int i = 0; i < pvals.length; i++) {
        Object value = resolveValueIfNecessary(beanName, mergedBeanDefinition,pvals[i].getName(), pvals[i].getValue());
        PropertyValue pv = new PropertyValue(pvals[i].getName(), value);
        // update mutable copy
        deepCopy.setPropertyValueAt(pv, i);
    }
    // set our (possibly massaged) deepCopy
    try {
        // synchronize if custom editors are registered
        // necessary because PropertyEditors are not thread-safe
        if (!getCustomEditors().isEmpty()) {
            synchronized (this) {
                bw.setPropertyValues(deepCopy);
            }
        }
        else {
            // 属性注入入口
            bw.setPropertyValues(deepCopy);
        }
    }
    catch (BeansException ex) {
        // improve the message by showing the context
        throw new BeanCreationException(mergedBeanDefinition.getResourceDescription(), beanName,"Error setting property values", ex);
    }
}
```

BeanWrapperImpl

```java
public void setPropertyValues(PropertyValues pvs) throws BeansException {
    setPropertyValues(pvs, false);
}

public void setPropertyValues(PropertyValues propertyValues, boolean ignoreUnknown) throws BeansException {
    List propertyAccessExceptions = new ArrayList();
    PropertyValue[] pvs = propertyValues.getPropertyValues();
    for (int i = 0; i < pvs.length; i++) {
        try {
            // This method may throw ReflectionException, which won't be caught
            // here, if there is a critical failure such as no matching field.
            // We can attempt to deal only with less serious exceptions.
            setPropertyValue(pvs[i]);
        }
        // fatal ReflectionExceptions will just be rethrown
        catch (NotWritablePropertyException ex) {
            if (!ignoreUnknown) {
                throw ex;
            }
            // otherwise, just ignore it and continue...
        }
        catch (TypeMismatchException ex) {
            propertyAccessExceptions.add(ex);
        }
        catch (MethodInvocationException ex) {
            propertyAccessExceptions.add(ex);
        }
    }

    // if we encountered individual exceptions, throw the composite exception
    if (!propertyAccessExceptions.isEmpty()) {
        Object[] paeArray = propertyAccessExceptions.toArray(new PropertyAccessException[propertyAccessExceptions.size()]);
        throw new PropertyAccessExceptionsException(this, (PropertyAccessException[]) paeArray);
    }
}

public void setPropertyValue(PropertyValue pv) throws BeansException {
    setPropertyValue(pv.getName(), pv.getValue());
}

public void setPropertyValue(String propertyName, Object value) throws BeansException {
    if (isNestedProperty(propertyName)) {
        try {
            // 和前述处理一致,如果是嵌套属性,找到最终的BeanWrapper,对其赋值
            BeanWrapper nestedBw = getBeanWrapperForPropertyPath(propertyName);
            nestedBw.setPropertyValue(new PropertyValue(getFinalPath(propertyName), value));
            return;
        }
        catch (NullValueInNestedPathException ex) {
            // let this through
            throw ex;
        }
        catch (FatalBeanException ex) {
            // error in the nested path
            throw new NotWritablePropertyException(propertyName, getWrappedClass(), ex);
        }
    }
    String[] tokens = getPropertyNameTokens(propertyName);
    setPropertyValue(tokens[0], tokens[1], tokens[2], value);
}

// propertyName为原始propertyName,可能包含[]''等字符,actutalName为褪去外层字符的实际属性名,key为collection类的index
private void setPropertyValue(String propertyName, String actualName, String key, Object value)throws BeansException {
    if (key != null) {
        Object propValue = getPropertyValue(actualName);
        if (propValue == null) {
            throw new FatalBeanException("Cannot access indexed value in property referenced in indexed property path '" + propertyName + "': returned null");
        }
        else if (propValue.getClass().isArray()) {
            Object[] array = (Object[]) propValue;
            array[Integer.parseInt(key)] = value;
        }
        else if (propValue instanceof List) {
            List list = (List) propValue;
            int index = Integer.parseInt(key);
            if (index < list.size()) {
                list.set(index, value);
            }
            else if (index >= list.size()) {
                for (int i = list.size(); i < index; i++) {
                    try {
                        list.add(null);
                    }
                    catch (NullPointerException ex) {
                        throw new FatalBeanException("Cannot set element with index " + index + " in List of size " +list.size() + ", accessed using property path '" + propertyName +"': List does not support filling up gaps with null elements");
                    }
                }
                list.add(value);
            }
        }
        else if (propValue instanceof Map) {
            Map map = (Map) propValue;
            map.put(key, value);
        }
        else {
            throw new FatalBeanException("Property referenced in indexed property path '" + propertyName +"' is neither an array nor a List nor a Map; returned value was [" + value + "]");
        }
    }
    else {
        if (!isWritableProperty(propertyName)) {
            throw new NotWritablePropertyException(propertyName, getWrappedClass());
        }
        PropertyDescriptor pd = getPropertyDescriptor(propertyName);
        Method writeMethod = pd.getWriteMethod();
        Object newValue = null;
        try {
            // old value may still be null
            newValue = doTypeConversionIfNecessary(propertyName, propertyName, null, value, pd.getPropertyType());
            if (pd.getPropertyType().isPrimitive() &&
                (newValue == null || "".equals(newValue))) {
                throw new IllegalArgumentException("Invalid value [" + value + "] for property '" + pd.getName() + "' of primitive type [" + pd.getPropertyType() + "]");
            }

            // 终于写了!
            writeMethod.invoke(this.object, new Object[] { newValue });
        }
        catch (InvocationTargetException ex) {
            // TODO could consider getting rid of PropertyChangeEvents as exception parameters
            // as they can never contain anything but null for the old value as we no longer
            // support event propagation.
            PropertyChangeEvent propertyChangeEvent = new PropertyChangeEvent(this.object, this.nestedPath + propertyName,  null, newValue);
            if (ex.getTargetException() instanceof ClassCastException) {
                throw new TypeMismatchException(propertyChangeEvent, pd.getPropertyType(), ex.getTargetException());
            }
            else {
                throw new MethodInvocationException(ex.getTargetException(), propertyChangeEvent);
            }
        }
        catch (IllegalAccessException ex) {
            throw new FatalBeanException("Illegal attempt to set property [" + value + "] threw exception", ex);
        }
        catch (IllegalArgumentException ex) {
            PropertyChangeEvent propertyChangeEvent = new PropertyChangeEvent(this.object, this.nestedPath + propertyName, null, newValue);
            throw new TypeMismatchException(propertyChangeEvent, pd.getPropertyType(), ex);
        }
    }
}
```



4.invokeInitMethods

```java
protected void invokeInitMethods(Object bean, String beanName, RootBeanDefinition mergedBeanDefinition)
			throws Exception {
    if (bean instanceof InitializingBean) {
        // afterPropertiesSet 调用时机早于initMethod
        ((InitializingBean) bean).afterPropertiesSet();
    }

    if (mergedBeanDefinition.getInitMethodName() != null) {
        bean.getClass().getMethod(mergedBeanDefinition.getInitMethodName(),
                                  null).invoke(bean, null);
    }
}
```
