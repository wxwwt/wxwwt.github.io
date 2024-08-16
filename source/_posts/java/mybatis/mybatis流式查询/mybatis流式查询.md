---
title: mybatis流式查询
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mybatis流式查询
---


# 导语：
有些时候我们所需要查询的数据量比较大，但是jvm内存又是有限制的，数据量过大会导致内存溢出。这个时候就可以使用流式查询，
数据一条条的返回，处理完一条在拿下一条数据，这样每次在内存里面的数据其实很小，不会导致内存溢出。
本文里面会讲到jdbc的流式查询和mybatis的流式查询。


## jdbc流式查询：
jdbc的流式查询需要在生成PreparedStatement的时候设置三个参数。如下：
```
PreparedStatement stmt = jdbcTemplate.getDataSource().getConnection().prepareStatement(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(Integer.MIN_VALUE);
```

主要使用到的是java.sql.Connection的prepareStatement方法。
```
PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency) throws SQLException;
```

resultSetType和resultSetConcurrency我们要分别设置为ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY。
还有就是fetchSize设置为Integer.MIN_VALUE，一开始比较疑惑为啥是这个值。后来发现代码里面对这个值其实是有特殊处理的。
这个是com.mysql.cj.jdbc.StatementImpl的setFetchSize方法。
```
@Override
   public void setFetchSize(int rows) throws SQLException {
       synchronized (checkClosed().getConnectionMutex()) {
           if (((rows < 0) && (rows != Integer.MIN_VALUE)) || ((this.maxRows > 0) && (rows > this.getMaxRows()))) {
               throw SQLError.createSQLException(Messages.getString("Statement.7"), MysqlErrorNumbers.SQL_STATE_ILLcom.mysql.cj.jdbc.StatementImpl的方法EGAL_ARGUMENT, getExceptionInterceptor());
           }

           this.query.setResultFetchSize(rows);
       }
   }
```

resultSetType，有以下三种
```
/**
     * The constant indicating the type for a <code>ResultSet</code> object
     * whose cursor may move only forward.
     * @since 1.2
     */
    int TYPE_FORWARD_ONLY = 1003;

    /**
     * The constant indicating the type for a <code>ResultSet</code> object
     * that is scrollable but generally not sensitive to changes to the data
     * that underlies the <code>ResultSet</code>.
     * @since 1.2
     */
    int TYPE_SCROLL_INSENSITIVE = 1004;

    /**
     * The constant indicating the type for a <code>ResultSet</code> object
     * that is scrollable and generally sensitive to changes to the data
     * that underlies the <code>ResultSet</code>.
     * @since 1.2
     */
    int TYPE_SCROLL_SENSITIVE = 1005;
```

```
stmt = conn.prepareStatement(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
stmt.setFetchSize(Integer.MIN_VALUE);
```

resultSetConcurrency有以下两种，流式查询要设置为只读的，数据不会被更新。
```
/**
     * The constant indicating the concurrency mode for a
     * <code>ResultSet</code> object that may NOT be updated.
     * @since 1.2
     */
    int CONCUR_READ_ONLY = 1007;

    /**
     * The constant indicating the concurrency mode for a
     * <code>ResultSet</code> object that may be updated.
     * @since 1.2
     */
    int CONCUR_UPDATABLE = 1008;
```

## mybatis流式查询：
mapper中的代码：
```
@Select("select * from xxx order by xx desc")
@Options(resultSetType = ResultSetType.FORWARD_ONLY, fetchSize = Integer.MIN_VALUE)
@ResultType(XxxObject.class)
void queryStreamResult(ResultHandler<XxxObject> handler);
```
在查询方法上加入注解@Options和@ResultType。设置参数不用多说和上面jdbc的底层是一样的，参数值也一样。
只不过设置了@ResultType告诉程序应该要返回什么类型的对象。还有这个ResultHandler其实就是Consumer的函数式接口用来处理每一条
返回的数据。

具体方法中的代码：
```
@Override
   public Boolean dealDataList() {
       mapper.queryStreamResult(resultContext -> {
           dealSingleData(resultContext.getResultObject().getUid());
       });
       return true;
   }
```
这里怎么使用每一条返回的数据只要在resultContext使用ResultObject就可以拿到上面mapper设置的XxxObject对象进行操作了。
