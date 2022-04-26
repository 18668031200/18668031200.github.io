---
title: monotonous
author: ygdxd
type: 转载
date: 2021-05-14 23:43:57
tags: spring
---

Spring Event 关于ContextStartedEvent 使用
=============================

 问题
----------------------------
 
 在spring boot 中定义event listener 实现ContextStartedEventListener<ContextStartedEvent> 监听上下文启动实现.发现项目启动后项目并不会打印出我们所需要的日志
 
{% codeblock ContextStartedEventListener.java lang:java %}

@Component
public class ContextStartedEventListener implements ApplicationListener<ContextStartedEvent> {

    private static final Logger log = LoggerFactory.getLogger(ContextStartedEventListener.class);

    @Override
    public void onApplicationEvent(ContextStartedEvent event) {
        log.info("ContextStartedEventListener : 项目启动");

    }
}
    
{% endcodeblock %}


解析
----------------------------

 打开spring 源码发现在AbstractApplicationContext#start() 方法中调用了此方法。
 ContextStoppedEvent同理也是stop方法。
 
 {% codeblock AbstractApplicationContext.java lang:java %}
 
 @Override
	public void start() {
		getLifecycleProcessor().start();
		publishEvent(new ContextStartedEvent(this));
	}

	@Override
	public void stop() {
		getLifecycleProcessor().stop();
		publishEvent(new ContextStoppedEvent(this));
	}
	
	{% endcodeblock %}
	
	定位到调用该方法的地方为DefaultLifecycleProcessor#start(). 很明显是在生命周期的处理器中调用。再次定位调用这个start方法的来源。
	
 {% codeblock AbstractApplicationContext.java lang:java %}
	 
	 
	 @Override
	public void start() {
		getLifecycleProcessor().start();
		publishEvent(new ContextStartedEvent(this));
	}	 
 	{% endcodeblock %}
 	
结论
-------------------------------------------
我们一般启动spring boot 是使用AbstractApplicationContext#refresh方法而不是start()方法，所以这里只能使用abstractApplicationContext 的start方法启动上下文是才会产生这个事件。


后记
--------------------------------------------
在DefaultLifecycleProcessor中使用spring boot 启动使用的是

{% codeblock DefaultLifecycleProcessor.java lang:java %}
@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	} 	
{% endcodeblock %}

他和刚刚的abstrctApplicationContext调用地start方法只有一个参数autoStartupOnly的区别。
这个参数为false时即我们abstrctApplicationContext.start时可以在我们的lifecycle bean 上使用注解@Phase 定义生命周期的阶段，从而自定义lifecycle bean 的start顺序。

这个注解还和timeoutPerShutdownPhase这个产数有关，timeoutPerShutdownPhase定义了lifecycle bean stop的超时时间
{% codeblock DefaultLifecycleProcessor.java lang:java %}
public void stop() {
			if (this.members.isEmpty()) {
				return;
			}
			this.members.sort(Collections.reverseOrder());
			CountDownLatch latch = new CountDownLatch(this.smartMemberCount);
			Set<String> countDownBeanNames = Collections.synchronizedSet(new LinkedHashSet<>());
			Set<String> lifecycleBeanNames = new HashSet<>(this.lifecycleBeans.keySet());
			for (LifecycleGroupMember member : this.members) {
				if (lifecycleBeanNames.contains(member.name)) {
					doStop(this.lifecycleBeans, member.name, latch, countDownBeanNames);
				}
				else if (member.bean instanceof SmartLifecycle) {
					// Already removed: must have been a dependent bean from another phase
					latch.countDown();
				}
			}
			try {
				latch.await(this.timeout, TimeUnit.MILLISECONDS);
			}
			catch (InterruptedException ex) {
				Thread.currentThread().interrupt();
			}
		}
   」

{% endcodeblock %}
