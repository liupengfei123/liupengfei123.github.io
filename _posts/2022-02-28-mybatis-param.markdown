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
                if (boundSql.hasAdditionalParameter(propertyName)) {
                    value = boundSql.getAdditionalParameter(propertyName);
                } else if (parameterObject == null) {
                    value = null;
                } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                    value = parameterObject;
                } else {
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
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
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    return result;
  }

```

通过上述代码中可以发现：在执行所有的语句之前都有先执行了 method.convertArgsToSqlCommandParam 语句，将参数封装成一个对象。

以上通过debug展示了设置参数的几个关键节点，下面我们就来通过源码来查看传入参数的具体实现及细节。

# 源码分析







# 总结

