





## IOC框架设计

- 接口`BeanFactory`到`HiberarchicalBeanFactory`, 再到`ConfigurableBeanFactory`, 是一条主要的BeanFactory设计路径. BeanFactory接口定义了基本的==IoC容器的规范==.  `HiberarchicalBeanFactory`接口在继承了BeanFactory的基本接口之后, 增加`getParentBeanFactory()`的接口功能, 使BeanFactory具备`双亲Ioc容器`的管理功能. `ConfigurableBeanFactory`接口继承BeanFactory的基本接口之后, 增加了`getParentBeanFactory()`的接口功能, 使得BeanFactory具备`双亲Ioc容器`功能.
- 以`ApplicationContext`应用上下文接口为核心的接口设计. 设计接口有, 从`BeanFactory`到`ListableBeanFactory`, 再到`ApplicationContext`, 再到WebApplicationContext或者`ConfigurableApplicationContext`的实现. 对于`ApplicationContext`接口, 通过继承`MessageSource`、`ResourceLoader`、`ApplicationEventPublisher`接口, 在BeanFactory简单Ioc容器的基础上添加==高级容器的特征==的支持.

![image-20190529233023240](http://ww4.sinaimg.cn/large/006tNc79gy1g3ilvd6rbzj31320si4cu.jpg)



### BeanFactory

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



## IoC容器的初始化过程

IoC容器的初始化由`AbstractApplicationContext.refresh()`方法来启动. 基本包括三个过程:  

1. [BeanDefinition的`Resource`定位](#Resouce定位)
2. [BeanDefinition载入](#BeanDefinition载入)
3. [Bean的注册](#BeanDefinition注册)

```java
// AbstractApplicationContext
@Override public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();
       // 在子类中启动refreshBeanFactory的地方 
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
			try {
				// 设置BeanFactory的后置处理.
				postProcessBeanFactory(beanFactory);
				// 调用BeanFactory的后处理器, 这些后处理器是在Bean定义中向容器注册的
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册Bean的后处理, 在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);
				// 对上下文消息源进行初始化
				initMessageSource();
				// 初始化上下文中的事件机制.
				initApplicationEventMulticaster();
				// 初始化其他特殊Bean
				onRefresh();
				// 检查监听Bean并且将这些Bean向容器注册.
				registerListeners();
         // 实例化所有的(non-lazy-init)单件 
				finishBeanFactoryInitialization(beanFactory);
				// 发布容器事件, 结束Refresh过程
				finishRefresh();
			}
  }
```

`DefaultListableBeanFactory`的IoC容器的**创建过程**:  

- 创建IoC配置文件的抽象`Resouce`, 这个抽象`Resource`包含了`BeanDefinition`的定义信息.
- 创建一个`BeanFactory`, 这里使用`DefaultListableBeanFactory`. 
- 创建一个加载BeanDefinition的**读取器**(`xxxBeanDefinitionReader`), 这里是`XmlBeanDefinitonReader`来载入XML文件形式的BeanDefintion, 通过一个回调配置给BeanFactory. 
- 从定义好的资源位置(`ResourceLoader`)读入配置信息, 具体解析过程由`XmlBeanDefinitionReader`来完成. 完成整个载入和注册Bean定义之后, 需要的Ioc容器就建立起来. 

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

> BeanDefinition读取器举例： AnnotatedBeanDefinitionReader; XmlBeanDefinitionReader；
>
> BeanDefinition读取器分别于xxxApplicationContext相对应， 通过给定的BeanFactory来读取BeanDefinition信息.



### Resouce定位

Resouce定位指的是`BeanDefinition`的资源定位, 由`ResouceLoader`统一的`Resouce`接口来完成.

```java
ClassPathResource res = new ClassPathResource("bean.xml");

public interface Resource extends InputStreamSource {
  URL getURL() throws IOException;
}
public class ClassPathResource extends Resource {}
public class ServletContextResource extends Resource {}
```

Spring已经为我们提供一系列不同`ResouceLoader`的实现类. `DefaultResourceLoader`是默认实现类

```java
public interface ResourceLoader {
  Resource getResource(String location);
  ClassLoader getClassLoader();
}

public class ServletContextResourceLoader extends DefaultResourceLoader {}
public class FileSystemResourceLoader extends DefaultResourceLoader {}
```

![image-20190530003852723](http://ww1.sinaimg.cn/large/006tNc79gy1g3iojd0fpjj30yi0u0gpm.jpg)

`FileSystemXmlApplicationContext`通过继承AbstractApplicationContext的基类`DefaultResourceLoader`, 具备了ResourceLoader载入以Resource定义的BeanDefinition的能力. 这个对BeanDefinition资源定位过程, 最初由`refresh() `来触发, 在`FileSystemXmlBeanFactory`的==构造函数==启动. 

![image-20190530004223606](http://ww1.sinaimg.cn/large/006tNc79gy1g3iojce8eej313s0pygoi.jpg)

`AbstractRefreshableApplicationContext`的`refreshBeanFactory()`被FileSystemXmlApplicationContext==构造函数==的`refresh`调用, 这个方法通过`createBeanFactory`构建一个**IoC容器**(DefautListableBeanFactory)供ApplicationContext使用. 

```java
// AbstractRefreshableApplicationContext
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
    throw new ApplicationContextException("I/O error parsing bean definition source for ");
  }
}

// 根据容器已有的双亲Ioc容器生成DefaultListableBeanFactory的双亲IoC容器
// 具体可以查看AbstractApplicationContext的实现
protected DefaultListableBeanFactory createBeanFactory() {
  return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}

// 加载BeanDefinition(代理给不同的BeanDefinitionReader具体实现类)
protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
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

####  BeanDefinitionReader

定义==标准==的`BeanDefinition`读取接口规范。每个具体类可以参考这种标准接口，还可以自定义。

```java
public interface BeanDefinitionReader {
  BeanDefinitionRegistry getRegistry();
  ResourceLoader getResourceLoader();
  int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;
  int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}

public class DefaultResourceLoader implements ResourceLoader {}
public class FileSystemResourceLoader extends DefaultResourceLoader {}
```

#### ResourceLoader

定位`Resource`的工具实现，`DefaultResourceLoader`是默认基类，提供了最基础的能力。

```java
@Overrid public Resource getResource(String location) {
  Assert.notNull(location, "Location must not be null");

  for (ProtocolResolver protocolResolver : this.protocolResolvers) {
    Resource resource = protocolResolver.resolve(location, this);
    if (resource != null) {
      return resource;
    }
  }

  if (location.startsWith("/")) {
    // 调用具体的getResource完成具体的Resource定位
    return getResourceByPath(location);
  }
  else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
    return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
  }
  else {
    try {
      // Try to parse the location as a URL...
      URL url = new URL(location);
      return new UrlResource(url);
    }
    catch (MalformedURLException ex) {
      return getResourceByPath(location);
    }
  }
}
```

### BeanDefinition载入

对于Ioc容器来说， 整个载入过程相当于把定义的`BeanDefinition`在IoC容器中转化成一个Spring内容表示的数据结构的过程。IoC容器对Bean的管理和依赖注入功能是通过对其持有的BeanDefinition进行各种相关操作来完成。这些`BeanDefinition`数据在IoC容器中通过`HashMap`来保持和维护。

![image-20190530134454614](http://ww3.sinaimg.cn/large/006tNc79ly1g3kc7ra53hj30pz0d7t9y.jpg)

上面的`loadBeanDefinition`在`AbstractXmlApplicationContext`中实现。创建`XMLBeanDefinitionReader`，调用`loadBeanDefinition(configLocation)`启动信息载入过程。

>  那么Spring的`BeanDefinition`是怎样按照Spring的Bean语义要求进行解析并转化为容器内部数据结构?，这个过程是在`registerBeanDefinition(doc， resource`)完成，`registerBeanDefinition`是定义在[BeanDefinitionRegistry](#BeanDefinition注册)中。





### BeanDefinition注册

用户定义的`BeanDefinition`信息在已经在IoC容器内建立自己的数据结构以及相应的数据表示，但此时这些数据还不能供IoC容器使用，需要在IoC容器中对这些`BeanDefinition`数据进行注册。以`DefaultListableBeanFactory`中， 是通过`HashMap`持有载入的BeanDefinition的。

```java
public class DefaultListableBeanFactory {
  private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
}
```

解析得到的beanDefinitionMap在`registerBeanDefinition`过程中发生。

![image-20190530140007825](http://ww4.sinaimg.cn/large/006tNc79ly1g3kc7pzjivj30i80dzgmj.jpg)

#### BeanDefinitionRegistry

`XMLBeanDefinition`通过给定的`BeanFactory`来载入`BeanDefinition`信息. `BeanFactory`实际上是通过实现`BeanDefinitionRegistry`来提供注册BeanDefinition能力.

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
  // 注册新BeanDefinition. 必须支持RootBeanDefinition和ChildDefinition.  
  void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
  void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
  
  //查询BeanDefinition, 通过BeanName.
  BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
  boolean containsBeanDefinition(String beanName);
}

```

```java
class DefaultListableBeanFactory {
  public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition){
    BeanDefinition oldBeanDefinition;
    oldBeanDefinition = this.beanDefinitionMap.get(beanName);
    ...
    this.beanDefinitionMap.put(beanName, beanDefinition);
    this.beanDefinitionNames.add(beanName);
    this.manualSingletonNames.remove(beanName);
    ...
  }
}
```



## IoC容器的依赖注入

<u>Bean的创建和对象依赖注入的过程中, 需要依据`BeanDefinition`中的信息来递归的完成依赖注入</u>. 递归是通过`getBean`为入口. 一个递归是在上下文体系中查找需要的`Bean`和`创建Bean`的递归调用; 另一个递归是在依赖注入时, 通过递归调用容器的`getBean`方法, 得到当前`Bean`的依赖`Bean`, 同时也触发对依赖Bean的创建和注入.  在对Bean的属性进行依赖注入时, 解析的过程也是一个递归的过程, 根据依赖关系, 一层一层地完成`Bean`的创建和注入, 直至最后完成当前Bean的创建.

> 首先注意到依赖注入的过程是用户第一次向IoC容器索要Bean时触发， 当然也有例外， 就是可以通过`lazy-init`属性让容器完成Bean的预实例化。

`DefaultListableBeanFactory`的基类`AbstractBeanFactory.getBean` 实现如下：

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
  ....
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
  
  // 通过调用createBean方法创建Singleton bean实例
  // 这里回调函数getObject，会在getSingleton中调用ObjectFactory的createBean
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
  ....创建prototype bean地方
  return (T)bean;
}

```

重点来说，`getBean`是依赖注入的起点, 之后会调用`createBean`, 通过createBean代码了解实现过程. Bean对象会根据BeanDefinition定义的要求生成. 在`AbstractAutoWireCapableBeanFactory`中实现`createBean`, createBean不但生成所需要的Bean, 还对Bean的属性进行初始化.

**AbstractAutowireCapableBeanFactory中的`createBean`**

![image-20190529171251023](http://ww3.sinaimg.cn/large/006tNc79gy1g3iayk9pcmj31gw0u0tgi.jpg)

的将诶的接

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

  // 这里判断需要创建的Bean是否可以实例化， 这个类是否可以通过类加载器载入
  Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  try {
	  //  如果Bean配置了PostProcessor, 那么这里需要返回一个Proxy	
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException("BeanPostProcessor before instantiation of bean failed");
  }
  
  // 实际创建Bean的方法
  Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  ...
  return beanInstance;
}

// 实际创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
  // BeanWrapper用来持有创建出来的Bean对象
  BeanWrapper instanceWrapper = null;
  // 如果是Singleton， 先把缓存中同名的Bean清楚
  if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
      // 这里是创建Bean的地方, 由createBeanInstance来完成
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
  Object exposedObject = bean;
  try {
    // 这里对Bean进行初始化, 依赖注入发生在这里, 这个exposedObject在初始化完以后会返回作为依赖注入完成的Bean
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
      exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
  }
  ...
  return exposedObject;
}
```

与依赖注入关系密切的方法有`createBeanInstance`和`popluateBean`.对象的生成有很多种不同的方式，可以通过`工厂方法`生成， 也可以通过容器的`autowire`特性生成，这些生成方式都是由相关的`BeanDefinition`来指定。

### CreateBean

**AbstractAutowireCapableBeanFactory.`createBeanInstance`**

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
  // 确定需要创建的Bean实例的类可以实例化
  Class<?> beanClass = resolveBeanClass(mbd, beanName);
  
  // 工厂方法对Bean进行实例化
  if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
  ...
  if (resolved) {
    if (autowireNecessary) {
      return autowireConstructor(beanName, mbd, null, null);
    }
    else {
      return instantiateBean(beanName, mbd);
    }
  }
  Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// 使用默认的构造函数对Bean进行实例化
		return instantiateBean(beanName, mbd);
}

protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
  try {
    Object beanInstance;
    final BeanFactory parent = this;
    if (System.getSecurityManager() != null) {
      beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
          return getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
      }, getAccessControlContext());
    }
    else {
      // 使用不同策略对Ban进行实例化，默认实例化策略是CglibSubclassingInstantiationStrategy
      // 使用CGLIB对Bean进行实例化
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
      mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
  }
}
```

`CGLIB`是一个常用的字节码生成类库, 提供一系列的API来提供生成和转换Java的字节码的功能.在`Spring AOP`中也使用CGLIB对Java的字节码进行增强.

```java
public interface InstantiationStrategy {
  	Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner)
			throws BeansException;
    Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner,
			Constructor<?> ctor, Object... args) throws BeansException;
    Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner,
			Object factoryBean, Method factoryMethod, Object... args) throws BeansException;
}

public class SimpleInstantiationStrategy implements InstantiationStrategy {
  public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (bd.getMethodOverrides().isEmpty()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
         // 取得指定的构造器或者对象的工厂方法来对Bean进行实例化 
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
								@Override
								public Constructor<?> run() throws Exception {
									return clazz.getDeclaredConstructor((Class[]) null);
								}
							});
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
       // 通过BeanUtils进行实例化
       // 内部通过ctor.newInstance(args)实现
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// 通过CGLIB来实例化对象
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
}
```

> `BeanUtils`是JavaBeans的反射工具类, 实例化 + 拷贝Bean属性值

`CglibSubclassingInstantiationStrategy`是默认实例化策略. `CGLIB`通过==动态==创建Class子类方式来增强类能力.

```java
// CglibSubclassingInstantiationStrategy
public Object instantiate(Constructor<?> ctor, Object... args) {
			Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
			Object instance;
			if (ctor == null) {
				instance = BeanUtils.instantiateClass(subclass);
			}
			else {
				try {
					Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
					instance = enhancedSubclassConstructor.newInstance(args);
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
							"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
				}
			}
			// SPR-10785: set callbacks directly on the instance instead of in the
			// enhanced class (via the Enhancer) in order to avoid memory leaks.
			Factory factory = (Factory) instance;
			factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
					new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
					new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
			return instance;
}
```

### PopluateBean

对Bean对象生成以后, 怎么样把这些Bean对象的==依赖关系设置==完, 完成整个依赖注入过程.

```java
// AbstractAutowireCapableBeanFactory
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
  PropertyValues pvs = mbd.getPropertyValues();
  ...
  // 开始进行依赖注入过程, 先处理autowire的注入
  if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
	}
  ...
  // 检查是否循环依赖
  if (needsDepCheck) {
      checkDependencies(beanName, mbd, filteredPds, pvs);
  }
  // 对属性进行注入
  applyPropertyValues(beanName, mbd, bw, pvs);
}

// 通过applyPropertyValues对具体属性进行解析然后注入
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
  ... 
  BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);
  for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
  ...
  // 依赖注入发生的地方, 在BeanWrapperImpl中完成.
  bw.setPropertyValues(new MutablePropertyValues(deepCopy));
}
```

### BeanDefinitionResolver

这里`BeanDefinitionResolver`来对BeanDefinition进行解析, 然后注入到`PropertyValue`.[BeanDefinitionValueResolver](http://www.itrator.top/category/设计之美/)类的名字所透露的， resolveValueIfNecessary方法是把Bean属性配置值解析成恰当的对象。如Xml中我们使用list标签，配置一个属性后，Spring内部实现会把涉及到的配置信息转化成内存中的List<Object>，从而跟业务直接对接, [详细解析源码解析点击](https://blog.csdn.net/zhuqiuhui/article/details/82391851).

> 内部实现和建模的角度看，Spring内部定义了如ManagedList、ManagedMap和ManagedSet等一系列。

```java
// BeanDefinitionValueResolver
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
		try {
       //RunTimeBeanReference是在BeanDefinition时根据配置生成的
			String refName = ref.getBeanName();
			refName = String.valueOf(doEvaluate(refName));
       // 如果ref是在双亲IoC容器中, 那就到双亲IoC容器中去获取.
			if (ref.isToParent()) {
				if (this.beanFactory.getParentBeanFactory() == null) {
					throw new BeanCreationException("xxxx")
				}
        
         // 如果依赖注入没有发生, 会触发一个getBean的过程.
				return this.beanFactory.getParentBeanFactory().getBean(refName);
			}
			else {
				Object bean = this.beanFactory.getBean(refName);
				this.beanFactory.registerDependentBean(refName, this.beanName);
				return bean;
			}
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					"Cannot resolve reference to beanName while setting "");
		}
}
```

### BeanWrapper

整个解析过程已经为依赖注入准备好了条件, 真正把Bean对象设置到它所以来的另一个Bean属性中去是在`BeanWrapper`的`setPropertyValue`中实现的.

----

```java
// BeanWrapper
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) {
  String propertyName = tokens.canonicalName;
	String actualName = tokens.actualName;
  PropertyTokenHolder getterTokens = new PropertyTokenHolder();
	getterTokens.canonicalName = tokens.canonicalName;
	getterTokens.actualName = tokens.actualName;
	getterTokens.keys = new String[tokens.keys.length - 1];
	System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);
  Object propValue;
  
  // getPropertyValue取得Bean中对注入对象的引用, 比如Array、List、Map、Set等
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

在Bean的创建和对象的依赖注入的过程中, 需要依据BeanDefinition中的信息来递归完成依赖注入. 这些递归都是以`getBean`为入口.一个递归在上下文体系中查找需要的Bean和创建Bean的递归调用, 同时也触发对依赖Bean的创建和注入.

## 容器相关特性的设计与实现

###  容器初始化和关闭过程

对于BeanFactory, 特别是ApplicationContext, 容器自身也有一个初始化和销毁关闭的过程. `AbstractApplicationContext.prepareBeanFactory`方法为容器配置了`ClassLoader`、`PropertyEditor`和`BeanPostProcessor`等, 从而为容器的启动做好必要的准备工作.

![image-20190530172445688](http://ww2.sinaimg.cn/large/006tNc79ly1g3kc7qqxcdj30ic0czt9g.jpg)

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

同样容器要关闭时, 也需要完成一系列的工作,  这些工作包括容器关闭的信号, 然后将Bean逐个关闭, 最后关闭容器自身.

### Bean的生命周期

!image-20190530141237022](/Users/likaihua/Library/Application Support/typora-user-images/image-20190530141237022.png)

![img](https://pic1.zhimg.com/80/v2-baaf7d50702f6d0935820b9415ff364c_hd.jpg)

### [容器aware支持](https://www.cnblogs.com/liaojie970/p/9055579.html)

容器的实现是通过Ioc管理Bean的生命周期来实现的. Spring Ioc容器在对Bean的生命周期进行管理时提供了Bean生命周期各个时间点的==回调.== 

- Bean实例的创建
- 为Bean实例设置属性
- 调用Bean的初始化方法
- 应用可以通过Ioc容器使用Bean
- 当容器关闭时, 调用Bean的销毁方法

### InitializeBean

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                public Object run() {
                    invokeAwareMethods(beanName, bean);
                    return null;
                }
            }, getAccessControlContext());
        }
        else {
            invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = **applyBeanPostProcessorsBeforeInitialization**(wrappedBean, beanName);
        }

        try {
            **invokeInitMethods**(beanName, wrappedBean, mbd);
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
        return wrappedBean;
   }


```

在调用Bean的初始化方法之前, 会调用一系列的`aware`接口实现, 把相关的`BeanName`、`BeanClassLoader`、`BeanFactory`注入到Bean中. 对应初始化可以在InitializingBean接口的afterPropertiesSet方法实现, 同样也是一个回调.

`invokeAwareMethods` 调用实现`Aware`接口方法，这里只针对三种Aware接口`BeanNameAware`，`BeanClassLoaderAware`，`BeanFactoryAware`，方法如下: 

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
            }
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
        }
 }
```



