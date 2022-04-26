---
title: java 自定义限流器
author: ygdxd
type: 原创
date: 2021-08-25 21:29:44
tags: java
categories: java
---

Java 自定义限流器


1.使用AtomicInetger cas实现
------------------------
{% codeblock Bucket.java lang:java %}

public class Bucket {

    private String name;

    private AtomicInteger count;

    public Bucket(String name) {
        this.name = name;
        count = new AtomicInteger(1);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public AtomicInteger getCount() {
        return count;
    }

    public void setCount(AtomicInteger count) {
        this.count = count;
    }
}


{% endcodeblock %}



{% codeblock RateLimiterAop.java lang:java %}

@Aspect
@Component
public class RateLimiterAop {

    private static final Logger LOGGER = LoggerFactory.getLogger(RateLimiterAop.class);

    private static final Map<String, Bucket> BUCKETS = new ConcurrentHashMap<>();

    private final Object mux = new Object();


    private static ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1);

    static {
        executorService.scheduleAtFixedRate(() -> {
            BUCKETS.values().stream().forEach(b -> {
                if (b.getCount().get() > 0) {
                    b.getCount().decrementAndGet();
                }
            });
        }, 0, 100, TimeUnit.MILLISECONDS);
    }

    @Around("execution(* com.ygdxd.ratelimiter.*.*(..))")
    public Object limit(ProceedingJoinPoint joinPoint) throws Throwable {
        //获取访问目标方法
        MethodSignature methodSignature = (MethodSignature)joinPoint.getSignature();
        Method targetMethod = methodSignature.getMethod();
        String apiName = targetMethod.getDeclaringClass().getName() + "." + targetMethod.getName();
        Bucket bucket = BUCKETS.get(apiName);
        if (bucket == null) {
            synchronized (mux) {
                BUCKETS.putIfAbsent(apiName, new Bucket(apiName));
            }
        } else {
            // 重试太多次 直接失败 也可以加锁
            if (!addCount(bucket, 0)) {
//                throw new RuntimeException("超出限制次数");
                LOGGER.error("失败");
            }
        }
        return joinPoint.proceed();
    }

    private boolean addCount(Bucket bucket, int i) {
        if (i > 3) {
            return false;
        }
        int count = bucket.getCount().get();
        if (count <= 10) {
            if (!bucket.getCount().compareAndSet(count, count + 1)){
                return addCount(bucket, i + 1);
            }
            return true;
        } else {
            return false;
        }
    }
}
{% endcodeblock %}

2.使用Semapphore实现(接口只能有几个线程，并不是每秒几个request)
------------------------

{% codeblock SemaphoreRateLimiterAop.java lang:java %}

@Aspect
@Component
public class SemaphoreRateLimiterAop {

    private static final Logger LOGGER = LoggerFactory.getLogger(SemaphoreRateLimiterAop.class);

    private static final Map<String, SemaphoreBucket> BUCKETS = new ConcurrentHashMap<>();

    private final Object mux = new Object();

    @Around("execution(* com.ygdxd.ratelimiter.*.*(..))")
    public Object limit(ProceedingJoinPoint joinPoint) throws Throwable {
        //获取访问目标方法
        MethodSignature methodSignature = (MethodSignature)joinPoint.getSignature();
        Method targetMethod = methodSignature.getMethod();
        String apiName = targetMethod.getDeclaringClass().getName() + "." + targetMethod.getName();
        SemaphoreBucket bucket = BUCKETS.get(apiName);
        if (bucket == null) {
            synchronized (mux) {
                bucket = new SemaphoreBucket(apiName);
                BUCKETS.putIfAbsent(apiName, bucket);
            }
        }
        boolean ack = false;
        try {
            ack = bucket.getCount().tryAcquire(1, TimeUnit.SECONDS);
            if (!ack) {
                throw new RuntimeException("限流了！");
            }
            return joinPoint.proceed();
        } catch (Exception e) {
            LOGGER.error("error:", e);
            throw new RuntimeException("失败！");
        }finally {
            if (ack) {
                bucket.getCount().release();
            }
        }
    }
}
{% endcodeblock %}

{% codeblock SemaphoreBucket.java lang:java %}

public class SemaphoreBucket {

    private String name;

    private Semaphore count;

    public SemaphoreBucket(String name) {
        this.name = name;
        count = new Semaphore(3);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Semaphore getCount() {
        return count;
    }

    public void setCount(Semaphore count) {
        this.count = count;
    }
}
{% endcodeblock %}

