

### Ioc容器实现



## Ioc容器设计

- 接口`BeanFactory`到`HiberarchicalBeanFactory`, 再到`ConfigurableBeanFactory`, 是一条主要的BeanFactory设计路径. BeanFactory接口定义了基本的==IoC容器的规范==.  `HiberarchicalBeanFactory`接口在继承了BeanFactory的基本接口之后, 增加`getParentBeanFactory()`的接口功能, 使BeanFactory具备`双亲Ioc容器`的管理功能. `ConfigurableBeanFactory`接口继承BeanFactory的基本接口之后, 增加了`getParentBeanFactory()`的接口功能, 使得BeanFactory具备`双亲Ioc容器`功能.
- 以`ApplicationContext`应用上下文接口为核心的接口设计. 设计接口有, 从`BeanFactory`到`ListableBeanFactory`, 再到`ApplicationContext`, 再到`WebApplicationContext`或者`ConfigurableApplicationContext`的实现. 对于`ApplicationContext`接口, 通过继承`MessageSource`、`ResourceLoader`、`ApplicationEventPublisher`接口, 在BeanFactory简单Ioc容器的基础上添加==高级容器的特征==的支持.

![image-20190529233023240](http://ww4.sinaimg.cn/large/006tNc79gy1g3ilvd6rbzj31320si4cu.jpg)



### BeanFactory的设计

`BeanFactory`接口提供最基础的Ioc容器的规范. `DefaultListableBeanFactory`作为一个默认的功能完整的Ioc容器, 包含了==基本IoC容器所有的重要功能==, 而各种`ApplicationContext`的实现是IoC容器的==高级表现形式==. 

```java
public interface BeanFactory {
  // Ioc容器Api主要方法, 通过该方法, 可以取得Ioc容器中管理的Bean, Bean的取得是通过指定名字来索引.
  Object getBean(String name) throws BeansException;
  <T> T getBean(String name, Class<T> requiredType) throws BeansException;

  // 判断容器是否含有指定名字的Bean
  boolean containsBean(String name);

  // 查询指定名字的Bean是否是Singleton类型的Bean.Singleton属性, 用户可以在BeanDefinition中指定.
  boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
  boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

  // 查询指定名字的Bean的Class类型是否是特定的Class类型.
  boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

  // 查询指定名字的Bean的所有别名, 这些别名都是用户定义在BeanDefinition.
  Class<?> getType(String name) throws NoSuchBeanDefinitionException;
  
}
```

`XmlBeanFactory`继承自`DefaultListableBeanFactory`, 增加**XML读取能力**. 其通过`XmlBeanDefinitionReader`对象来读取那些以XML形式的`BeanDefinition`的信息. 

![image-20190529235519330](http://ww3.sinaimg.cn/large/006tNc79gy1g3iojdhdlsj31a00hcwhc.jpg)



### IoC容器的初始化过程

IoC容器的初始化由`refresh()`方法来启动. 基本包括三个过程:  

1. [BeanDefinition的`Resource`定位](#Resouce定位)
2. BeanDefinition的载入
3. Bean的注册

`DefaultListableBeanFactory`的IoC容器的**基本过程**:  

- 创建IoC配置文件的抽象`Resouce`, 这个抽象`Resource`包含了`BeanDefinition`的定义信息.
- 创建一个`BeanFactory`, 这里使用`DefaultListableBeanFactory`. 
- 创建一个加载BeanDefinition的**读取器**(`xxxBeanDefinitionReader`), 这里是`XmlBeanDefinitonReader`来载入XML文件形式的BeanDefintion, 通过一个回调配置给BeanFactory. 
- 从定义好的资源位置(`ResourceLoader`)读入配置信息, 具体解析过程由`XmlBeXmlBeanDefinitionReader`来完成. 完成整个载入和注册Bean定义之后, 需要的Ioc容器就建立起来. 

```java
// 创建Ioc容器步骤
ClassPathResource res = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);

// 以FileSystemXmlApplicationContext容器为例
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
      // IoC容器初始化正式启动.
			refresh();
		}
	}
```

### Resouce定位

Resouce定位指的是`BeanDefinition`的资源定位, 由`ResouceLoader`统一的`Resouce`接口来完成.

```java
ClassPathResource res = new ClassPathResource("bean.xml");

public interface Resource extends InputStreamSource {
  URL getURL() throws IOException;
}
public class ClassPathResource implement Resource {
}
```

Spring已经为我们提供一系列不同`Resouce`的读取器的实现类.

```java
public interface ResourceLoader {
  Resource getResource(String location);
  ClassLoader getClassLoader();
}
```

![image-20190530003852723](http://ww1.sinaimg.cn/large/006tNc79gy1g3iojd0fpjj30yi0u0gpm.jpg)

`FileSystemXmlApplicationContext`通过继承AbstractApplicationContext的基类`DefaultResourceLoader`, 具备了ResourceLoader载入以Resource定义的BeanDefinition的能力. 这个对BeanDefinition资源定位过程, 最初由`refresh() `来触发, 在`FileSystemXmlBeanFactory`的**构造函数**启动. 

![image-20190530004223606](http://ww1.sinaimg.cn/large/006tNc79gy1g3iojce8eej313s0pygoi.jpg)

`AstractorRefreshAppplicationContext`的`refreshBeanFactory()`被FileSystemXmlApplicationContext构造函数的`refresh`调用, 这个方法通过`createBeanFactory`构建一个IoC容器(DefautListableBeanFactory)供ApplicationContext使用. 

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  // 创建并设置持有DefaultListableBeanFactory, 同时调用loadBeanDefinition载入BeanDefinition的信息.
  try {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}

// 根据容器已有的双亲Ioc容器生成DefaultListableBeanFactory的双亲IoC容器
protected DefaultListableBeanFactory createBeanFactory() {
  return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}

protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
    // 载入BeanDefinition信息
		loadBeanDefinitions(beanDefinitionReader);
	}

// 通过BeanDefnitionReader载入BeanDefinition信息
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
  Resource[] configResources = getConfigResources();
  if (configResources != null) {
    reader.loadBeanDefinitions(configResources);
  }
  String[] configLocations = getConfigLocations();
  if (configLocations != null) {
    reader.loadBeanDefinitions(configLocations);
  }
}
```





### IoC容器的依赖注入

Bean的创建和对象依赖注入的过程中, 需要依据`BeanDefinition`中的信息来递归的完成依赖注入. 递归是通过`getBean`为入口. 一个递归是在上下文体系中查找需要的`Bean`和`创建Bean`的递归调用; 另一个递归是在依赖注入时, 通过递归调用容器的`getBean`方法, 得到当前`Bean`的依赖`Bean`, 同时也触发对依赖Bean的创建和注入.  在对Bean的属性进行依赖注入时, 解析的过程也是一个递归的过程, 根据依赖关系, 一层一层地完成`Bean`的创建和注入, 直至最后完成当前Bean的创建.

![image-20190529171251023](http://ww3.sinaimg.cn/large/006tNc79gy1g3iayk9pcmj31gw0u0tgi.jpg)



**DefaultListableBeanFactory的基类AbstractBeanFactory.getBean**

---

```java
public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
		return doGetBean(name, requiredType, args, false);
}

protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
  // 先从缓存中去的Bean, 处理那些已经被创建过的单例模式的Bean, 对这种Bean的请求不需要重复创建
  Object sharedInstance = getSingleton(beanName);
  //....
  // 检查是否能在当前BeanFactory中取得需要的Bean. 如果在当前的工厂中取不到, 则到双亲BeanFactory中取, 如果当前的双亲工厂取不到, 则顺着双亲BeanFactory链一直向上查找
  BeanFactory parentBeanFactory = getParentBeanFactory();
  // 根据BeanName获得BeanDefinition
  final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
  // 获取当前Bean依赖的所有Bean, 这样会触发getBean的递归调用, 直至取到一个没有任何依赖的Bean为止
  String[] dependsOn = mbd.getDependsOn();
  if (dependsOn != null) {
    for (String dep : dependsOn) {
      if (isDependent(beanName, dep)) {
        throw new BeanCreationException("Circular depends-on");
      }
      registerDependentBean(dep, beanName);
      // 递归调用
      getBean(dep);
    }
	}
  
  // Create bean instance.
  if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>(){
      @Override
      public Object getObject() throws BeansException {
        try {
          // 创建Bean
          return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
          throw ex;
        }
      }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
  }
  ....
}

```



`getBean`是依赖注入的起点, 之后会调用`createBean`, 通过createBean代码了解实现过程. Bean对象会根据BeanDefinition定义的要求生成. 在`AbstractAutoWireCapableBeanFactory`中实现createBean, createBean不但生成所需要的Bean, 还对Bean的属性进行初始化.

**AbstractAutowireCapableBeanFactory中的createBean**

---

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) {
  RootBeanDefinition mbdToUse = mbd;
  // 这里判断需要创建的Bean是否可以实例化, 这个类是否可以通过类加载器载入
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null){
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}
  
  Object bean = resolveBeforeInstantiation(beanName, mbdToUse);try {
	//  如果Bean配置了PostProcessor, 那么这里需要返回一个Proxy	
  Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException("BeanPostProcessor before instantiation of bean failed");
  }
  
  // 创建Bean
  Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  ...
  return beanInstance;
}

// 实际创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
  // BeanWrapper用来持有创建出来的Bean对象
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
      // 这里是创建Bean的地方, 有createBeanInstance来完成
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
  Object exposedObject = bean;
  try {
    // 这里对Bean进行初始化, 依赖注入发生在这里, 这个exposedObject在初始化完以后会返回作为依赖注入完成的Bean
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
      exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
  }...
  return exposedObject;
}
```

与依赖注入关系密切的方法有`createBeanInstance`和`popluateBean`.



**BeanWrapper完成Bean的属性值注入**

----

```java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) {
  String propertyName = tokens.canonicalName;
	String actualName = tokens.actualName;
  PropertyTokenHolder getterTokens = new PropertyTokenHolder();
	getterTokens.canonicalName = tokens.canonicalName;
	getterTokens.actualName = tokens.actualName;
	getterTokens.keys = new String[tokens.keys.length - 1];
	System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);
  Object propValue;
	propValue = getPropertyValue(getterTokens);
  if (propValue.getClass().isArray()) {
    // ....
    Array.set(propValue, arrayIndex, convertedValue);
  } else if (propValue instanceof List) {
		list.add(convertedValue);
  } else if (propValue instanceof Map) {
    map.put(convertedMapKey, convertedMapValue);
  } else {
     PropertyHandler ph = getLocalPropertyHandler(actualName);
     ph.setValue(this.wrappedObject, valueToApply);
  }
```



### 容器相关特性的设计与实现

容器的实现是通过Ioc管理Bean的生命周期来实现的. Spring Ioc容器在对Bean的生命周期进行管理时提供了Bean生命周期各个时间点的回调. 

- Bean实例的创建
- 为Bean实例设置属性
- 调用Bean的初始化方法
- 应用可以通过Ioc容器使用Bean
- 当容器关闭时, 调用Bean的销毁方法

