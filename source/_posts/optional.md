---
title: Java Optional的用法
author: ygdxd
type: 原创
date: 2021-08-10 16:15:01
tags: jdk
categories: jdk
---


关于Optional,平时项目用的不多.笔者觉得Java 引入Optional 主要用于解决参数时传递一些可能为空对象时产生的空指针问题.

1.使用Optional的一些问题 filter
----------------

bad code:
{% codeblock Demo.java lang:java %}
optional.filter(s -> StrUtil.isNotEmpty(s.getName())).orElseThrow(() -> new IllegalParamException());
optional.filter(s -> StrUtil.isNotEmpty(s.getStartDate())).orElseThrow(() -> new IllegalParamException());
optional.filter(s -> StrUtil.isNotEmpty(s.getEndDate())).orElseThrow(() -> new IllegalParamException());
optional.filter(s -> s.getStartDate().compareTo(s.getEndDate()) <= 0).orElseThrow(() -> new IllegalParamException());
{% endcodeblock %}
影响整体代码的阅读和可扩展性,使用if语句判断或者spring validator 来进行参数校验可能会更好.

good code
{% codeblock Demo.java lang:java %}
optional.filter(s -> StrUtil.isNotEmpty(s.getGroupName()))
        .filter(s -> StrUtil.isNotEmpty(s.getStartDate()))
        .filter(s -> StrUtil.isNotEmpty(s.getEndDate()))
        .filter(s -> s.getStartDate().compareTo(s.getEndDate()) <= 0)
        .orElseThrow(() -> new IllegalParamException());
{% endcodeblock %}       
缺点是不能根据具体的参数返回对应的错误信息.

{% codeblock Optional.java lang:java %}
public Optional<T> filter(Predicate<? super T> predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
 }
{% endcodeblock %}
filter 先判断当前Optional是否有值,如果存在就使用predicate判断是否符合条件,如果不通过就返回一个空的Optional.

2.使用Optional的一些问题 map
----------------

往往在对接第三方服务时会使用多层嵌套结构

比如 Company -> Department -> Team -> User

bad code
{% codeblock Demo.java lang:java %}
if (company != null) {
   Department dep = company.getDepartment();
   if (dep != null) {
     ...
   }
}
{% endcodeblock %}
good code
{% codeblock Demo.java lang:java %}
Optional.ofNullable(Company)
    .map(c -> c.getDepartment())
    .map(d -> d.getTeam())
    .map(t -> t.getUser())
    .orElseThrow(NoSuchElementException::new);
  {% endcodeblock %}  
map 和 filter 一样,只有当前optional里的值不为空才会去执行对应的函数表达式,原理和if一样但是更简洁.

{% codeblock Optional.java lang:java %}
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }
 {% endcodeblock %}   
3. orElse 和 orElseGet的区别

{% codeblock Optional.java lang:java %}
public T orElse(T other) {
        return value != null ? value : other;
    }
    
public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
{% endcodeblock %}
2个方法的参数不同,如果传入的参数为固定的值或者对象,那么2个方法的处理和返回没有任何区别.
但是当传入的是表达式时,orElse会首先执行表达式获得结果,然后入栈.
而 orElseGet 的参数是 Supplier,所以直接入栈,然后在调用 other.get()的时候才会被触发.
明显在特殊情况下后者的性能更好.


