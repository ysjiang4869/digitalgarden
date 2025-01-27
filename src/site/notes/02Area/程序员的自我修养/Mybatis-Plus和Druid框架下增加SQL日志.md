---
{"dg-publish":true,"permalink":"/02-area//mybatis-plus-druid-sql/","title":"Mybatis-Plus和Druid框架下增加SQL日志","tags":["Database"],"created":"2025-01-27 10:57","updated":"2025-01-27 11:15"}
---


# Mybatis-Plus和Druid框架下增加SQL日志

## 基础情况

### 问题

sharding-jdbc输出的日志太繁琐，而且不可用定制日志格式、不能动态调整是否输出。

期望能够动态修改sql日志级别，select语句默认不打印日志，仅在排查问题时候输出。

### 项目信息

采用druid作为连接池，使用Mybatis-Plus作为orm框架，采用sharding-jdbc进行分库分表。

由此，数据的访问层次为：查询->mybatis-plus->mybatis>sharding-jdbc datasource->druid

因此，在任何一个层级都可以拦截进行日志输出

调研存在如下拦截方式：
- mybatis-plus: InnerInterceptor拦截器，其实是mybatis-plus的插件开发方式
- mybatis: interceptor拦截器，提供整个生命周期的拦截
- sharding-jdbc：未找到合理的拦截方式
- druid: filter过滤器

## 解决方案

基于以上拦截方式，产生下述方案，最后我选用了mybatis的interceptor的方案，其实我更推荐mybatis-plus的InnerInterceptor，我的项目基本都是基于此orm框架的，但是也不排除后续复杂的场景会使用xml文件，所以暂且这样。

### mybatis-plus

自定义`InnerInterceptor`，这里不再给出具体代码，参考后文`MyBatisSqlUtil`即可实现拿到sql打印了。

然后配置如下

```java
@Bean  
public MybatisPlusInterceptor mybatisPlusInterceptor() {  
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));//如果配置多个插件,切记分页最后添加  
    //interceptor.addInnerInterceptor(new PaginationInnerInterceptor()); 如果有多数据源可以不配具体类型 否则都建议配上具体的DbType  
    return interceptor;  
}
```

#### DataChangeRecorderInnerInterceptor

看官方文档的插件，其中，自带的`DataChangeRecorderInnerInterceptor` 可以用来进行审计，也可以实现打印SQL的效果。但是不能打印select请求，同时会多进行一次查询获取旧的数据，普通日志打印不推荐开启。

### mybatis

自定义拦截器如下

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.util.Properties;

/**
 * sql打印拦截器
 */
@Intercepts({
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
    @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})
})
@Slf4j
public class SqlInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 计算这一次SQL执行钱后的时间，统计一下执行耗时
        long startTime = System.currentTimeMillis();
        Object proceed = invocation.proceed();
        long endTime = System.currentTimeMillis();

        String printSql = null;
        boolean isSelect = false;
        try {
            // 获取到BoundSql以及Configuration对象
            // BoundSql 对象存储了一条具体的 SQL 语句及其相关参数信息。
            // Configuration 对象保存了 MyBatis 框架运行时所有的配置信息
            MappedStatement statement = (MappedStatement) invocation.getArgs()[0];
            SqlCommandType sct = statement.getSqlCommandType();
            isSelect = sct == SqlCommandType.SELECT;
            Object parameter = null;
            if (invocation.getArgs().length > 1) {
                parameter = invocation.getArgs()[1];
            }
            Configuration configuration = statement.getConfiguration();
            BoundSql boundSql = statement.getBoundSql(parameter);
            printSql = MyBatisSqlUtil.generateSql(configuration, boundSql);
        } catch (Exception exception) {
            log.error("获取sql异常", exception);
        } finally {
            // 拼接日志打印过程
            long costTime = endTime - startTime;
            if (isSelect) {
                log.debug("\n running SQL cost: {}ms, actual SQL: {}", costTime, printSql);
            } else {
                log.info("\n running SQL cost: {}ms, actual SQL: {}", costTime, printSql);
            }
        }
        return proceed;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        // 可以通过properties配置插件参数
    }
}
```

其中，通用的sql拼接方法

```java
import lombok.experimental.UtilityClass;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.type.TypeHandlerRegistry;
import org.springframework.util.ObjectUtils;

import java.text.DateFormat;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.regex.Matcher;

/**
 * mybatis sql工具
 */
@UtilityClass
public class MyBatisSqlUtil {


    /**
     * 获得实际执行的sql
     */
    public static String generateSql(Configuration configuration, BoundSql boundSql) {
        // 获取参数对象
        Object parameterObject = boundSql.getParameterObject();
        // 获取参数映射
        List<ParameterMapping> params = boundSql.getParameterMappings();
        // 获取到执行的SQL
        String sql = boundSql.getSql();
        // SQL中多个空格使用一个空格代替
        sql = sql.replaceAll("\\s+", " ");
        if (!ObjectUtils.isEmpty(params) && !ObjectUtils.isEmpty(parameterObject)) {
            // TypeHandlerRegistry 是 MyBatis 用来管理 TypeHandler 的注册器。TypeHandler 用于在 Java 类型和 JDBC 类型之间进行转换
            TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
            // 如果参数对象的类型有对应的 TypeHandler，则使用 TypeHandler 进行处理
            if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                sql = sql.replaceFirst("\\?", Matcher.quoteReplacement(getParameterValue(parameterObject)));
            } else {
                // 否则，逐个处理参数映射
                for (ParameterMapping param : params) {
                    // 获取参数的属性名
                    String propertyName = param.getProperty();
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    // 检查对象中是否存在该属性的 getter 方法，如果存在就取出来进行替换
                    if (metaObject.hasGetter(propertyName)) {
                        Object obj = metaObject.getValue(propertyName);
                        sql = sql.replaceFirst("\\?", Matcher.quoteReplacement(getParameterValue(obj)));
                        // 检查 BoundSql 对象中是否存在附加参数。附加参数可能是在动态 SQL 处理中生成的，有的话就进行替换
                    } else if (boundSql.hasAdditionalParameter(propertyName)) {
                        Object obj = boundSql.getAdditionalParameter(propertyName);
                        sql = sql.replaceFirst("\\?", Matcher.quoteReplacement(getParameterValue(obj)));
                    } else {
                        // 如果都没有，说明SQL匹配不上，带上“缺失”方便找问题
                        sql = sql.replaceFirst("\\?", "缺失");
                    }
                }
            }
        }
        return sql;
    }

    private static String getParameterValue(Object object) {
        String value = "";
        if (object instanceof String) {
            value = "'" + object + "'";
        } else if (object instanceof Date) {
            DateFormat format = DateFormat.getDateTimeInstance(DateFormat.DEFAULT, DateFormat.DEFAULT, Locale.CHINA);
            value = "'" + format.format((Date) object) + "'";
        } else if (!ObjectUtils.isEmpty(object)) {
            value = object.toString();
        }
        return value;
    }
}
```

最后，在`SqlSessionFactory`中设置plugin

```java
@Bean("sqlSessionFactory")  
@Primary  
public SqlSessionFactory sqlSessionFactory(MybatisPlusInterceptor mybatisPlusInterceptor) throws Exception {  
    MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();  
    //... your code
    bean.setPlugins(mybatisPlusInterceptor,new Test());  
    return bean.getObject();  
}
```

### druid filter

#### 自定义filter

下面的方案没有完全实现，因为不确定要实现哪些方法来打印，没有深入研究这些方法还

```java
package com.cmcc.services.robot.business.user.config.db;  
  
import com.alibaba.druid.filter.FilterAdapter;  
import com.alibaba.druid.filter.FilterChain;  
import com.alibaba.druid.proxy.jdbc.ResultSetProxy;  
import com.alibaba.druid.proxy.jdbc.StatementProxy;  
  
import java.sql.SQLException;  
  
public class SqlPrintFilter extends FilterAdapter {  
  
    @Override  
    public ResultSetProxy statement_executeQuery(FilterChain chain, StatementProxy statement, String sql) throws SQLException {  
        // 打印 SQL 语句  
        System.out.println("Executing Query SQL: " + sql);  
        return super.statement_executeQuery(chain, statement, sql);  
    }  
  
    @Override  
    public int statement_executeUpdate(FilterChain chain, StatementProxy statement, String sql) throws SQLException {  
        // 打印 SQL 语句  
        System.out.println("Executing Update SQL: " + sql);  
        return super.statement_executeUpdate(chain, statement, sql);  
    }  
  
    @Override  
    public boolean statement_execute(FilterChain chain, StatementProxy statement, String sql) throws SQLException {  
        // 打印 SQL 语句  
        System.out.println("Executing SQL: " + sql);  
        return super.statement_execute(chain, statement, sql);  
    }  
  
}
```

使用

```java
datasource.setProxyFilters(Lists.newArrayList(statFilter(),new SqlPrintFilter()));
```

#### 使用自带的LogFilter

自带的多个`LogFilter`实现，示例配置如下

```java
Slf4jLogFilter logFilter = new Slf4jLogFilter();  
logFilter.setStatementCreateAfterLogEnabled(false);  
logFilter.setStatementLogEnabled(true);  
logFilter.setStatementExecutableSqlLogEnable(true);  
logFilter.setStatementLogErrorEnabled(true);  
logFilter.setResultSetLogEnabled(false);  
```

缺点是，默认打印的日志太长了，不受控制，也不能调整格式
