---
title: SpringBean
author: ygdxd
type: 原创
date: 2021-07-23 19:14:20
tags: spring
categories: spring
---

关于spring bean的实例化和初始化流程的一些记录

1.BeanDedifinition的beanClass设置
-------------------
当想要去创建bean时,是想创建BeanDedifinition,而在BeanDedifinition创建完之后,Bean实例化之前BeanDedifinition的beanClass由String类型转成对应Bean的class类型,而ClassLoader则是由当前线程的ClassLoader(AppClassLoader)进行加载.

那beanClass为什么是String类型的.
{% codeblock AbstractBeanDefinition.java lang:java %}
@Override
	public void setBeanClassName(@Nullable String beanClassName) {
		this.beanClass = beanClassName;
	}
{% endcodeblock %}
这里设置BeanClassName时默认设置beanClass为类名.那我们在实例化bean时必须先把beanClass转换成对应的class对象.
对应的设置的方法为AbstractBeanFactory的resolveBeanClass方法.在方法里又调用了doResolveBeanClass方法,里面先获取ClassLoader,之后调用AbstractBeanDefinition的resolveBeanClass方法.在这个方法里通过ClassLoader它设置我们beanClass.

{% codeblock AbstractBeanFactory.java lang:java %}
@Nullable
	private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)
			throws ClassNotFoundException {

		ClassLoader beanClassLoader = getBeanClassLoader();
		ClassLoader dynamicLoader = beanClassLoader;
		boolean freshResolve = false;
		// 省略部分代码
		....
		return mbd.resolveBeanClass(beanClassLoader);
	}
	
{% endcodeblock %}

在这里设置beanClass.
{% codeblock AbstractBeanDefinition.java lang:java %}
@Nullable
	public Class<?> resolveBeanClass(@Nullable ClassLoader classLoader) throws ClassNotFoundException {
		String className = getBeanClassName();
		if (className == null) {
			return null;
		}
		Class<?> resolvedClass = ClassUtils.forName(className, classLoader);
		this.beanClass = resolvedClass;
		return resolvedClass;
	}

{% endcodeblock %}


2. spring aware 接口回调
-------------------

首先创建bean

{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
			
			// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		// 如果是单例就不用重复实例化了
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//其他步骤
		...
		
		try {
		    // 设置bean的属性 
			populateBean(beanName, mbd, instanceWrapper);
			// 初始化bean
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
			
			
}			
{% endcodeblock %}

在AbstractAutowireCapableBeanFactory的initializeBean方法中实现了对spring中Aware的回调

{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}

protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
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
{% endcodeblock %}

这个方法中首先判断安全,然后调用invokeAwareMethods 这个方法
{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}

private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
{% endcodeblock %}

这个方法依次调用了BeanNameAware, BeanClassLoaderAware, BeanFactoryAware.这里并没有调用ApplicationContextAware,因为这里是初始化bean相关的回调. ApplicationContextAware回调在applyBeanPostProcessorsBeforeInitialization方法中进行.

再来看下applyBeanPostProcessorsBeforeInitialization

{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}
@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
{% endcodeblock %}

通过获取beanfactory里面的BeanPostProcessors 然后调用postProcessBeforeInitialization.如果我们的BeanPostProcessors中包含了ApplicationContextAwareProcessor(这是一个内部类),那么spring便会实现ApplicationContextAware回调.

{% codeblock ApplicationContextAwareProcessor.java lang:java %}
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
{% endcodeblock %}

这里面回调了关于application context上下文的Aware接口
{% codeblock ApplicationContextAwareProcessor.java lang:java %}
private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
{% endcodeblock %}


我们再回到initializeBean方法 ,在applyBeanPostProcessorsBeforeInitialization方法之后调用了invokeInitMethods方法.里面先调用了实现InitializingBean的afterPropertiesSet方法,之后如果bean有InitMethod就调用它.
{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
{% endcodeblock %}

最后applyBeanPostProcessorsAfterInitialization主要回调BeanPostProcessor.postProcessAfterInitialization方法,例如ApplicationListener Bean初始化之后,ApplicationListenerDetector在此阶段将其添加到applicationContext.


{% codeblock AbstractAutowireCapableBeanFactory.java lang:java %}
@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
{% endcodeblock %}
