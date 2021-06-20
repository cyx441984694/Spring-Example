Spring框架理解

在Spring中，我们听到最多的名词是Bean。那么其实Spring就是面向Bean的编程。
Spring里，改变了常规的依赖关系，把依赖关系转为用配置文件。 而这种机制就在一个IoC容器管理。换句话来说，IoC容器中管理这Bean的对象及他们之间的依赖关系。
Spring中核心组件是Core, Context和Bean。 Core理解为IoC容器，Context是协调员，负责创造建立跟协调Bean之间的关系。

1. 构建Bean工厂

Ioc容易实际上是Context组件结合bean组件和core组件共同构建了一个Bean关系网。
构建的入口在ApplicationContext类中(`org.springframework.context.support.ApplicationContext`)
```
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				// Initialize message source for this context.
				initMessageSource();
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();
				// Initialize other special beans in specific context subclasses.
				onRefresh();
				// Check for listener beans and register them.
				registerListeners();
				// Instantiate all remaining (non-lazy-init) singletons. 
				finishBeanFactoryInitialization(beanFactory);
				// Last step: publish corresponding event.
				finishRefresh();
			}
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();
				// Reset 'active' flag.
				cancelRefresh(ex);
				// Propagate exception to caller.
				throw ex;
			}
		}
	}
```
实际完成以下目标: </br>
* 构建BeanFactory，当已存在时就刷新配置
* 注册可能感兴趣的事件(event) 
* 创建Bean实例对象
* 触发被监听的事件

在AbstractRefreshableApplicationContext中可以看到refreshBeanFactory方法。其中可以知道两点:
默认的BeanFactory原始对象是DefaultListableBeanFactory； 在其中会`loadBeanDefinitions`然后跳转到`XmlBeanDefinitionReader` 解析Bean的定义
```.env
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
```

```.env
loadBeanDefinitions
	/**
	 * Load bean definitions into the given bean factory, typically through
	 * delegating to one or more bean definition readers.
	 * @param beanFactory the bean factory to load bean definitions into
	 * @throws BeansException if parsing of the bean definitions failed
	 * @throws IOException if loading of bean definition files failed
	 * @see org.springframework.beans.factory.support.PropertiesBeanDefinitionReader
	 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
	 */
	protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
			throws BeansException, IOException;
```

2. 创建Bean实例

Bean的实例化是在上面的refresh()中的finishBeanFactoryInitialization方法开始的。其中PreInstantiateSingletons方法就是解释Bean的实例化:
```.env
DefaultListableBeanFactory.java
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
```
在这里会出现一个FactoryBean，可以产生Bean实例的bean。创建Bean实例就是按照上面的代码的过程。
其中的判断包括:
* 是否可实例化
* 是否是FactoryBean
* 是否是Eager Init
* 是否父类对象存在
* Bean对象是否有依赖关系
* Bean对象的类型

而如何建立Bean对象实例之间的关系，简单来说，会先初始化新建BeanWrapperImpl对象，从而可以调用并获取PropertyValue，利用resolver相关的方法来进行调用并利用反射机制把属性的值注入当前对象的属性中。
我觉得在书中这个比喻得很好，比较简短地说一下: </br>

把IoC容器比喻一个箱子，里面会有球跟制造球的球模具，还有造模具的机器。 </br>
* 造模具的机器 = BeanFactory
* 球模 = Bean
* 球 = Bean实例
* BeanFactoryPostProcessor = 在球模被造起来，可以对其修改
* InitializingBean跟DisposableBean是开始造球和结束简单的预备扫尾工作。

3. 参考资料

* 《深入分析Java Web技术内幕》 * 许令波
* https://www.tutorialspoint.com/spring/spring_hello_world_example.htm
