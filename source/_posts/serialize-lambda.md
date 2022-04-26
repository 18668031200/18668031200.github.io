---
title: SerializeLambda
author: ygdxd
type: 原创
date: 2021-09-06 20:43:13
tags: java
categori: java
---

在mybatis plus 查询中想根据某个字段排序,但是字段是字符串类型,百度了下mysql提供了几种方法。

比如根据code进行排序
```mysql
#1.
ORDER BY
	`code` * 1 ASC
#2.
ORDER BY
	`code` + 0 ASC
#1.
ORDER BY
	CAST(`code` AS DECIMAL) ASC
```
然而项目中代码使用了LambdaQueryWrapper 里面指定了使用lambda来指定排序的列，并不能根据想要的特殊情况进行排序。

本来想着entity里加一个数据库不存在的字段 然后设置这个字段的名称为 code+0 但是mybatis plus 不会把@Tablefiled
中exist为false的放进去

AbstractLambdaWrapper 获取不到对应的列信息
COLUMN_CACHE_MAP 是根据TableInfo 创建的.在initTableFields时会过滤
```java
private synchronized static TableInfo initTableInfo(Configuration configuration, String currentNamespace, Class<?> clazz) {
        /* 没有获取到缓存信息,则初始化 */
        TableInfo tableInfo = new TableInfo(clazz);
        tableInfo.setCurrentNamespace(currentNamespace);
        tableInfo.setConfiguration(configuration);
        GlobalConfig globalConfig = GlobalConfigUtils.getGlobalConfig(configuration);

        /* 初始化表名相关 */
        final String[] excludeProperty = initTableName(clazz, globalConfig, tableInfo);

        List<String> excludePropertyList = excludeProperty != null && excludeProperty.length > 0 ? Arrays.asList(excludeProperty) : Collections.emptyList();

        /* 初始化字段相关 */
        initTableFields(clazz, globalConfig, tableInfo, excludePropertyList);

        /* 自动构建 resultMap */
        tableInfo.initResultMapIfNeed();

        /* 缓存 lambda */
        LambdaUtils.installCache(tableInfo);
        return tableInfo;
    }
```
```java
public static List<Field> getAllFields(Class<?> clazz) {
        List<Field> fieldList = ReflectionKit.getFieldList(ClassUtils.getUserClass(clazz));
        return fieldList.stream()
            .filter(field -> {
                /* 过滤注解非表字段属性 */
                TableField tableField = field.getAnnotation(TableField.class);
                return (tableField == null || tableField.exist());
            }).collect(toList());
    }
```

查看了下如何获取列名的源码

```java
public static <T> SerializedLambda resolve(SFunction<T, ?> func) {
Class<?> clazz = func.getClass();
String name = clazz.getName();
// 使用WeakReference缓存
return Optional.ofNullable(FUNC_CACHE.get(name))
.map(WeakReference::get)
.orElseGet(() -> {
    // 这里根据lambda 获取SerializedLambda
    SerializedLambda lambda = SerializedLambda.resolve(func);
FUNC_CACHE.put(name, new WeakReference<>(lambda));
return lambda;
});
}
```

```java
 /**
     * 通过反序列化转换 lambda 表达式，该方法只能序列化 lambda 表达式，不能序列化接口实现或者正常非 lambda 写法的对象
     *
     * @param lambda lambda对象
     * @return 返回解析后的 SerializedLambda
     */
    public static SerializedLambda resolve(SFunction<?, ?> lambda) {
        if (!lambda.getClass().isSynthetic()) {
            throw ExceptionUtils.mpe("该方法仅能传入 lambda 表达式产生的合成类");
        }
        // 使用字节码进行读取
        try (ObjectInputStream objIn = new ObjectInputStream(new ByteArrayInputStream(SerializationUtils.serialize(lambda))) {
            @Override
            protected Class<?> resolveClass(ObjectStreamClass objectStreamClass) throws IOException, ClassNotFoundException {
                Class<?> clazz;
                try {
                    clazz = ClassUtils.toClassConfident(objectStreamClass.getName());
                } catch (Exception ex) {
                    clazz = super.resolveClass(objectStreamClass);
                }
                return clazz == java.lang.invoke.SerializedLambda.class ? SerializedLambda.class : clazz;
            }
        }) {
            return (SerializedLambda) objIn.readObject();
        } catch (ClassNotFoundException | IOException e) {
            throw ExceptionUtils.mpe("This is impossible to happen", e);
        }
    }
```

```java
/**
     * 获取 SerializedLambda 对应的列信息，从 lambda 表达式中推测实体类
     * <p>
     * 如果获取不到列信息，那么本次条件组装将会失败
     *
     * @param lambda     lambda 表达式
     * @param onlyColumn 如果是，结果: "name", 如果否： "name" as "name"
     * @return 列
     * @throws com.baomidou.mybatisplus.core.exceptions.MybatisPlusException 获取不到列信息时抛出异常
     * @see SerializedLambda#getImplClass()
     * @see SerializedLambda#getImplMethodName()
     */
    private String getColumn(SerializedLambda lambda, boolean onlyColumn) {
        // 获取类名
        Class<?> aClass = lambda.getInstantiatedType();
        // 初始化 主要是缓存一下列信息
        tryInitCache(aClass);
        String fieldName = PropertyNamer.methodToProperty(lambda.getImplMethodName());
        ColumnCache columnCache = getColumnCache(fieldName, aClass);
        return onlyColumn ? columnCache.getColumn() : columnCache.getColumnSelect();
    }
```

SerializedLambda
=====================
这里面发现能把lambda表达式进行序列化,然后获取相关lambda的信息.

![SerializedLambda.java](/images/serialized_lambda.jpg)


