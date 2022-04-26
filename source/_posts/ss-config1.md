---
title: spring security 1 配置类加载
author: ygdxd
type: 原创
date: 2019-04-22 16:00:25
tags: spring security
---


##spring security 配置类加载


####WebSecurityConfigurerAdapter

一般我们在使用spring security作为我们安全验证的时候经常会编写配置类继承WebSecurityConfigurerAdapter，通过重写其中的config()类来自定义自己的安全验证流程。而HttpSecurity类则是其中非常重要的一个配置类，通过它你可以集成其他第三方的生态来满足自己的业务需求。

<!-- more -->

####HttpSecurity

在HttpSecurity 中我们可以看到很多配置方法比如

{% codeblock HttpSecurit.java lang:java %}

public OpenIDLoginConfigurer<HttpSecurity> openidLogin() throws Exception {
		return getOrApply(new OpenIDLoginConfigurer<>());
	}
	
...

public HeadersConfigurer<HttpSecurity> headers() throws Exception {
		return getOrApply(new HeadersConfigurer<>());
	}

{% endcodeblock %}

很明显它们都调用了getOrAppley方法

{% codeblock HttpSecurit.java lang:java %}

private <C extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity>> C getOrApply(
			C configurer) throws Exception {
		//从已加载中的配置类根据class获取
		C existingConfig = (C) getConfigurer(configurer.getClass());
		//不为空的或就返回已加载类
		if (existingConfig != null) {
			return existingConfig;
		}
		return apply(configurer);
	}

{% endcodeblock %}

再看apply这个方法

{% codeblock AbstractConfiguredSecurityBuilder lang:java %}

public <C extends SecurityConfigurerAdapter<O, B>> C apply(C configurer)
			throws Exception {
		//添加后置处理器
		configurer.addObjectPostProcessor(objectPostProcessor);
		//设置builder
		configurer.setBuilder((B) this);
		//把这个配置类添加到配置类集合中
		add(configurer);
		return configurer;
	}

{% endcodeblock %}

首先objectPostProcessor 应该是ObjectPostProcessorConfiguration这个配置类中的AutowireBeanFactoryObjectPostProcessor实例，它管理了一系列的SmartInitializingSingleton的afterSingletonsInstantiated方法和DisposableBean的destroy方法，以确保他们被调用。

configurer.setBuilder((B) this); 则是把builder类的引用放到配置类中（SecurityConfigurerAdapter子类）这样配置类就可以通过getbuilder()方法来实现一系列操作。

最后configer会被放入一个集合中通过doBuild()方法来进行加载初始化

{% codeblock AbstractConfiguredSecurityBuilder lang:java %}


protected final O doBuild() throws Exception {

		//加锁
		synchronized (configurers) {
			buildState = BuildState.INITIALIZING;

			beforeInit();
			init();

			buildState = BuildState.CONFIGURING;

			beforeConfigure();
			configure();

			buildState = BuildState.BUILDING;

			O result = performBuild();

			buildState = BuildState.BUILT;

			return result;
		}
	}

{% endcodeblock %}


各种配置类有各自的实现，这样ss就可以扩展安全验证的机制了。
