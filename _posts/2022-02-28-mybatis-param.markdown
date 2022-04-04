---
layout:     page
title:      "从使用到源码：MyBatis 参数"
subtitle:   "MyBatis SQL参数填充原理"
date:       2022-02-28
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - mybatis
    - 原理分析
---

# 概述

我们知道 **MyBatis**是一个持久层框架，它是对jdbc的操作数据库的过程进行了封装。 所以通过原生jbdc的操作过程映射在MyBatis上可以加快我们对其源码和原理的了解。

![图1：JDBC连接池流程图]({{site.baseurl}}/img/in-post/mybatis-param/1646052089096.png)

而本章要讲解的就是上图中的 **传入参数** 在mybatis中是如何实现的？

# 传入流程

写个简单的测试用例用于debug。

```java
@Test
public void test2() {
    SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();
    try (SqlSession openSession = sqlSessionFactory.openSession()) {
        EmployeeMapper mapper = openSession.getMapper(EmployeeMapper.class);
        // select * from employee  where id = #{id}
        Employee employee = mapper.getEmpById(1);
        System.out.println(employee);
    }
}
```

由于Mybatis只是封装了JDBC的操作，所以jdbc中的代码最终还是会执行的，所以我们可以在数据库驱动PreparedStatement 类的setXXX 方法打上断点，然后通过debug方式执行测试用例。

![1646054020880]({{site.baseurl}}/img/in-post/mybatis-param/1646054020880.png)

这样我们就可以得到传入参数的方法调用栈。

向栈顶方法逐个观察其栈帧，其中在 IntegerTypeHandler#setNonNullParameter 方法中。

```java
public class IntegerTypeHandler extends BaseTypeHandler<Integer> {
  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
      throws SQLException {
    // setInt 方法即为 JDBC 的传入参数的方法。
    ps.setInt(i, parameter);
  }
}
```

在查看 IntegerTypeHandler 包路径下的其他类。

![1646053436559]({{site.baseurl}}/img/in-post/mybatis-param/1646053436559.png)

在查看 IntegerTypeHandler 的父类。

```java
public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T> {
  @Override
  public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) {
    if (parameter == null) {
      if (jdbcType == null) {
        throw new TypeException("XXXX");
      }
      ps.setNull(i, jdbcType.TYPE_CODE);
    } else {
      setNonNullParameter(ps, i, parameter, jdbcType);
    }
  }
  public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
}
```

通过多态将不同类型的设置参数方法语句抽取成不同的类，以方便下面使用策略模式。

在查看 DefaultParameterHandler#setParameters 方法。

```java
public void setParameters(PreparedStatement ps) {
    // 该集合中就存储着 sql语句中从左到右的各个参数信息对象。
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
        for (int i = 0; i < parameterMappings.size(); i++) {
            ParameterMapping parameterMapping = parameterMappings.get(i);
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                // 实际设置参数的值
                Object value;
                // 参数名词
                String propertyName = parameterMapping.getProperty();
			   // .... 获取参数代码
                
                // 策略模式 获取传入对应类型的执行器
                TypeHandler typeHandler = parameterMapping.getTypeHandler();
                JdbcType jdbcType = parameterMapping.getJdbcType();
                if (value == null && jdbcType == null) {
                    jdbcType = configuration.getJdbcTypeForNull();
                }
                try {
                    // 向 PreparedStatement 中设置参数
                    typeHandler.setParameter(ps, i + 1, value, jdbcType);
                } catch (TypeException | SQLException e) {
                    throw new TypeException(e);
                }
            }
        }
    }
}
```

在 MapperMethod#execute 中

```java
// 其中 args 数组为传入的参数
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      //....  SELECT、UPDATE、DELETE、FLUSH 等处理逻辑。
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
  }

```

通过上述代码中可以发现：在执行所有的语句之前都有先执行了 method.convertArgsToSqlCommandParam 语句，将参数封装成一个对象。

以上通过debug展示了设置参数的几个关键节点，下面我们就来通过源码来查看传入参数的具体实现及细节。

# 源码分析

## 传入参数处理

由上述可知对传入参数处理是在 **MethodSignature#convertArgsToSqlCommandParam** 方法中，而该方法中又是调用的 **ParamNameResolver#getNamedParams**方法

```java
public Object getNamedParams(Object[] args) { 
    // names 中在构建 ParamNameResolver 对象是创建，将参数列表按照按照索引顺序存在其中
    //aMethod(@Param("M") int a, @Param("N") int b) -> ((0, "M"), (1, "N"))
	//aMethod(int a, @Param("N") int b) -> ((0, "0"), (1, "N"))
	//aMethod(int a, RowBounds rb, int b) -> ((0, "0"), (2, "1"))  #参数类型为 RowBounds、ResultHandler排除
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        // 接口方法没有设置参数则直接返回空
        return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
        // 如果参数没有使用 @Param 注解，并且只有一个参数
        Object value = args[names.firstKey()];
        return wrapToMapIfCollection(value, useActualParamName ? names.get(0) : null);
    } else {
        // 将参数包装成Map。
        final Map<String, Object> param = new ParamMap<>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            param.put(entry.getValue(), args[entry.getKey()]);
          
            final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
            if (!names.containsValue(genericParamName)) {
                // 对于没有使用 @param注解的参数，额外添加通用key (param + index) 其中index相对于names内的顺序。
                // generic param names (param1, param2, ...)
                param.put(genericParamName, args[entry.getKey()]);
            }
            i++;
        }
        return param;
    }
}

// 如果对象是集合或者数组，这就将其包装成map
public static Object wrapToMapIfCollection(Object object, String actualParamName) {
    if (object instanceof Collection) {
        ParamMap<Object> map = new ParamMap<>();
        map.put("collection", object);
        if (object instanceof List) {
            map.put("list", object);
        }
        Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
        return map;
    } else if (object != null && object.getClass().isArray()) {
        ParamMap<Object> map = new ParamMap<>();
        map.put("array", object);
        Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
        return map;
    }
    return object;
}
```

## sql占位符处理

在 DefaultParameterHandler#setParameters 有使用了一个变量 parameterMappings，该变量就是在配置文件中sql语句的占位符信息集合。

那么这个数据是如果构建出来的呢？

在  SqlSourceBuilder#parse 方法中

```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
   
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql;
    // 解析sql语句将其中的占位符替换为 ‘？’，记录占位符数据
    if (configuration.isShrinkWhitespacesInSql()) {
        sql = parser.parse(removeExtraWhitespaces(originalSql));
    } else {
        sql = parser.parse(originalSql);
    }
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}
```

在 SqlSourceBuilder.ParameterMappingTokenHandler 方法中

```java
// 当解析到 #{XXXX} 时，会将 XXXX 传入该方法中。
// 按照顺序将占位符进行记录，并根据配置设置参数信息保存在 ParameterMapping对象中
@Override
public String handleToken(String content) {
    parameterMappings.add(buildParameterMapping(content));
    return "?";
}

private ParameterMapping buildParameterMapping(String content) {
    // .... 进行解析占位符设置，来构建 ParameterMapping对象
}
```

以上是为#{}预编译占位符的形式，而使用${}直接拼接sql的形式则不是按照上面的流程。

使用${}的时候则使用了 DynamicSqlSource 类对sql进行动态解析，（动态sql也是使用该类到时候在详细说明）。

在 DynamicSqlSource#getBoundSql 方法中。

```java
@Override
public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    // rootSqlNode 为配置文件中sql语句节点信息
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    // 在解析的过程中 遇到 bind 节点（BindHandler） 会将数据存放在 binds 中。
    context.getBindings().forEach(boundSql::setAdditionalParameter);
    return boundSql;
}
```

在解析过程中经过 TextSqlNode 类节点。

```java
@Override
public boolean apply(DynamicContext context) {
    GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
    context.appendSql(parser.parse(text));
    return true;
}

private GenericTokenParser createParser(TokenHandler handler) {
    // 通过 BindingTokenParser 解析 ${}
    return new GenericTokenParser("${", "}", handler);
}
```

在 BindingTokenParser 类的 handleToken 方法中

```java
@Override
public String handleToken(String content) {
    Object parameter = context.getBindings().get("_parameter");
    if (parameter == null) {
        context.getBindings().put("value", null);
    } else if (SimpleTypeRegistry.isSimpleType(parameter.getClass())) {
        context.getBindings().put("value", parameter);
    }
    // 从参数列表中获取数据，在创建 DynamicContext 对象的时候将 parameterObject 参数列表存在 _parameter 中。
    // 在测试的时候发现当只存在一个参数时，content 的key值在 binds中明显没有匹配的，但是还是返回正确的数据，后发现原因在 ContextMap（binds类型）中的get方法
    Object value = OgnlCache.getValue(content, context.getBindings());
    String srtValue = value == null ? "" : String.valueOf(value); // issue #274 return "" instead of "null"
    checkInjection(srtValue);
    // 在这里就将 ${} 在sql语句中替换为实际的数据。
    return srtValue;
}
```

在 DynamicContext 类中。

```java
private final ContextMap bindings;
public DynamicContext(Configuration configuration, Object parameterObject) {
    if (parameterObject != null && !(parameterObject instanceof Map)) {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        boolean existsTypeHandler = configuration.getTypeHandlerRegistry().hasTypeHandler(parameterObject.getClass());
        bindings = new ContextMap(metaObject, existsTypeHandler);
    } else {
        bindings = new ContextMap(null, false);
    }
    //PARAMETER_OBJECT_KEY 为 _parameter
    bindings.put(PARAMETER_OBJECT_KEY, parameterObject);
}
static class ContextMap extends HashMap<String, Object> {
    private static final long serialVersionUID = 2977601501966151582L;
    private final MetaObject parameterMetaObject;

    @Override
    public Object get(Object key) {
        String strKey = (String) key;
        if (super.containsKey(strKey)) {
            return super.get(strKey);
        }
        if (parameterMetaObject == null) {
            return null;
        }
        if (fallbackParameterObject && !parameterMetaObject.hasGetter(strKey)) {
            // 当 参数只有一个并且存在类型处理器的时候 可以不用管 key 的内容
            return parameterMetaObject.getOriginalObject();
        } else {
            // issue #61 do not modify the context when reading
            return parameterMetaObject.getValue(strKey);
        }
    }
}
```

## 获取参数

经过以上的铺垫将要可以进行设置参数了，但是还差一步，就是将实际的参数列与sql中的参数名词配对，也就是现在需要讲的获取参数数据。

在 DefaultParameterHandler#setParameters 方法的片段

```java
Object value;
String propertyName = parameterMapping.getProperty();
// 获取通过binds节点设置的数据， DynamicSqlSource#getBoundSql 方法中设置
if (boundSql.hasAdditionalParameter(propertyName)) {
    value = boundSql.getAdditionalParameter(propertyName);
} else if (parameterObject == null) {
    value = null;
} else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
    // 如果该参数类型有被注册过类型处理中，则可以直接处理，也说明了参数列表中只存在一个参数
    value = parameterObject;
} else {
    MetaObject metaObject = configuration.newMetaObject(parameterObject);
    // 通过 XXX.YYY.ZZZ 的形式，获取参数，具体细节可以通过查看mybatis源码中对MetaObject类的单元测试用例。
    value = metaObject.getValue(propertyName);
}
```

## 设置参数

到这里就到了最后一步了，

在 DefaultParameterHandler#setParameters 方法的片段

```java
// 在处理占位符时，SqlSourceBuilder.ParameterMappingTokenHandler 中设置
TypeHandler typeHandler = parameterMapping.getTypeHandler();
JdbcType jdbcType = parameterMapping.getJdbcType();
if (value == null && jdbcType == null) {
    jdbcType = configuration.getJdbcTypeForNull();
}
try {
    typeHandler.setParameter(ps, i + 1, value, jdbcType);
} catch (TypeException | SQLException e) {
    throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
}
```



```java
@Override
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    if (parameter == null) {
        if (jdbcType == null) {
            throw new TypeException(XXXXXXXXX);
        }
        try {
            ps.setNull(i, jdbcType.TYPE_CODE);
        } catch (SQLException e) {
            throw new TypeException(XXXXXXXXX);
        }
    } else {
        try {
            setNonNullParameter(ps, i, parameter, jdbcType);
        } catch (Exception e) {
            throw new TypeException(XXXXXXXXX);
        }
    }
}
```

在StringTypeHandler类中

```java
@Override
public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, parameter);
}
```

