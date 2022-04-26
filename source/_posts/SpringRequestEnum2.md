---
title: 关于spring boot在请求参数中使用枚举(enums)的几种方式(2)
author: ygdxd
type: 原创
date: 2021-07-21 21:04:11
tags: spring
categories: spring 
---

在日常开发中，我们都会在请求中使用某些枚举类型，这样不仅能够防止恶意客户端输入一些错误值，提高程序的健壮性，还可以提高代码的可读性和可维护性。当时当我们在request 参数中添加枚举后，spring boot并不会帮助我们自动解析成枚举，这就需要我们能够自己配置了。下面介绍几种日常开发中会用的方式。

3.使用@JsonCreator
-------------------

一把项目中,我们使用都是POST方法使用body来接收对象,在对象中我们定义了枚举参数,但是当我们传入json字符串反序列化时，项目会出现错误400.JSON parse error: Cannot deserialize value of type.

那这个错误是怎么产生的呢？
我们通过debug发现在RequestMappingHandlerAdapter.java这个类的invokeHandlerMethod方法中将请求转换为调用我们写的接口的.
而这个方法最终会调用InvocableHandlerMethod.java的getMethodArgumentValues来获取其中的参数.那如何来获取呢，便是通过里面的resolver集合.当我们使用@RequestBody来接收请求参数是,系统对应的选择RequestResponseBodyMethodProcessor来处理.
前面我们的GET请求是使用RequestParamMethodArgumentResolver来处理的.
那RequestResponseBodyMethodProcessor是怎么处理的,它也和RequestParamMethodArgumentResolver一样是使用converter来处理,但是使用的是HttpMessageConverter,因为他是读取消息体来获取参数的.

{% codeblock AbstractMessageConverterMethodArgumentResolver.java lang:java %}
for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					else {
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					break;
				}
			}
{% endcodeblock %}

这里我们看到程序使用了一个for循环来尝试过去处理,而我们输入的是json字符串那么最终采用的便是我们一直用的MappingJackson2HttpMessageConverter.这个converter在反序列化成枚举是便会产生异常.

那解决方法便是尝试让我们的枚举类支持反序列化方式和增加我们自己定义的converter.这里方便起见我们可以直接用@JsonCreater来让我们的枚举支持反序列化.


首先在我们的项目的IntegerRequestEnum这个枚举类中增加静态方法

{% codeblock IntegerRequestEnum.java lang:java %}
@JsonCreator
public static IntegerRequestEnum decode(Integer source) {
        for (IntegerRequestEnum e : IntegerRequestEnum.values()) {
            if (e.getCode().equals(source)) {
                return e;
            }
        }
        return A;
    }
{% endcodeblock %}

请求类对象:
{% codeblock EnumRequest.java lang:java %}
public class EnumRequest {

private IntegerRequestEnum ir;

    public IntegerRequestEnum getIr() {
        return ir;
    }

    public void setIr(IntegerRequestEnum ir) {
        this.ir = ir;
    }
}
{% endcodeblock %}

接着定义我们的接口
{% codeblock EnumController.java lang:java %}
@PostMapping("/post")
    public ResponseEntity<?> postEnumRequest(@RequestBody EnumRequest A) {
        return ResponseEntity.ok(A.getIr().getCode());
    }
    
 {% endcodeblock %}
 
 使用POSTMAN访问成功返回我们需要的值.
 
 
4.使用JsonDeserializer(不推荐)
-----------------

最简单的方法但是枚举类一多配置起来比较复杂

定义我们的Deserializer
{% codeblock IntegerEnumDeserializer.java lang:java %}
public static class IntegerEnumDeserializer extends JsonDeserializer<IntegerRequestEnum> {

        @Override
        public IntegerRequestEnum deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
            return IntegerRequestEnum.decode(jsonParser.getIntValue());
        }
    }

{% endcodeblock %}

然后在我们Request对象里的枚举属性上使用注解@JsonDeserialize(using = IntegerEnumDeserializer.class)


最后
------------

阿里在开发手册中不推荐返回值使用枚举,认为枚举在可扩展性上存在不足.

