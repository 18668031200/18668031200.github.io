---
title: 关于spring boot在请求参数中使用枚举(enums)的几种方式 (1)
author: ygdxd
type: 原创
date: 2021-07-20 21:43:05
tags: spring
categories: spring
---


在日常开发中，我们都会在请求中使用某些枚举类型，这样不仅能够防止恶意客户端输入一些错误值，提高程序的健壮性，还可以提高代码的可读性和可维护性。当时当我们在request 参数中添加枚举后，spring boot并不会帮助我们自动解析成枚举，这就需要我们能够自己配置了。


1.直接使用字符串
--------------

先定义一个接收输入的枚举类

{% codeblock RequestEnum.java lang:java %}

public enum RequestEnum {
    /**
     * A
     */
    A("A"),
    /**
     * B
     */
    B("B");

    RequestEnum(String code) {
        this.code = code;
    }

    private String code;

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }
}

{% endcodeblock %}

接着定义一个接收GET请求的接口

{% codeblock EnumController.java lang:java %}
@GetMapping
    public ResponseEntity<?> getEnumRequest(@RequestParam RequestEnum A) {
        return ResponseEntity.ok(A.getCode());
    }
    
{% endcodeblock %}

这个时候我们启动项目，在浏览器输入
http://localhost:8080/**?A=A

这个时候能够看到浏览器显示一个字母A 说明spring boot 能够把我们的字符串成功转换成枚举RequestEnum。

2.使用Integer或者其他类型
================

我们再定义一个枚举
{% codeblock IntegerRequestEnum.java lang:java %}
public enum IntegerRequestEnum {

    /**
     * A
     */
    A(1),
    /**
     * B
     */
    B(2);

    IntegerRequestEnum(Integer code) {
        this.code = code;
    }

    private Integer code;

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }
}

{% endcodeblock %}

接着编写接收这个枚举的接口

{% codeblock EnumController.java lang:java %}
@GetMapping("int")
    public ResponseEntity<?> getIntegerEnumRequest(@RequestParam IntegerRequestEnum A) {
        return ResponseEntity.ok(A.getCode());
    }
    
{% endcodeblock %}

这个时候我们在浏览器输入
http://localhost:8080/**/int?A=2

发现浏览器返回浏览错误信息

There was an unexpected error (type=Bad Request, status=400).

3.解析
=================
为什么String类型可以直接解析成枚举而字符串不能呢？
我们看下他是怎么解析的。
{% codeblock InvocableHandlerMethod.java lang:java %}
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
			
			if (ObjectUtils.isEmpty(getMethodParameters())) {
			return EMPTY_ARGS;
		}
		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			// 省略部分代码
			try {
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				//错误处理			}
		}
		return args;
	
			
	}

{% endcodeblock %}

在InvocableHandlerMethod.java 这个类里spring 通过反射帮我们获取了方法接收的参数的类型，然后通过resolvers去查找处理(里面使用map保存,增加速度).最终调用了RequestParamMethodArgumentResolver的父类AbstractNamedValueMethodArgumentResolver来处理。


{% codeblock AbstractNamedValueMethodArgumentResolver.java lang:java %}
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
			
			// 获取值
			Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
			// @RequestParam 的值处理
			if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveStringValue(namedValueInfo.defaultValue);
			}
			// 在这里optional是可以为空的
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveStringValue(namedValueInfo.defaultValue);
		}
			
		// 在这里处理成我们想要的枚举类型
		arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
				
	}

{% endcodeblock %}

在binder.convertIfNecessary调用方法中我们最终调用了TypeConverterDelegate这个代理类的convertIfNecessary这个方法.这里面最终执行了如下代码
{% codeblock TypeConverterDelegate.java lang:java %}
if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
				try {
					return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
				}
				catch (ConversionFailedException ex) {
					// fallback to default conversion logic below
					conversionAttemptEx = ex;
				}
			}

{% endcodeblock %}

在这里最终会产生异常.
这里面会调用String->Enum 这个converter,它是用了enum的构造方法进行转化.但是我们的IntegerRequestEnum的构造方法是Integer.
另外我们通过debug还能看到conversionService的converters里一共缓存了124个converter.

![](images/converters.png)

所以如果我们需要解析我们的enum就需要在这个conterters里增加我们的自定义的converter.
{% codeblock WebConfig.java lang:java %}
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(String.class, IntegerRequestEnum.class, new StringToIntegerEnumConverter());
    }
}

{% endcodeblock %}

增加配置当我们再次访问http://localhost:8080/**/int?A=2之后页面就能正常返回.

