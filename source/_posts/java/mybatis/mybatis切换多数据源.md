---
title: mybatis切换多数据源
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mybatis
---
### 引语
有时候我们需要切换多个数据源,来使用不同的数据,这时候我们可以使用mybatis的DataSourceTransactionManager来实现数据源的切换.
下面我们一起来看下来一个简单的实现方式.

### 步骤:
1.首先我们jdbc.properties(名字不一定是这个,代指保存数据库连接信息的配置文件)里面有多个数据源,一个老的一个新的
```
jdbc.driver:com.mysql.Driver
jdbc.url:xxx
jdbc.username:xxx
jdbc.password:xxx

old.jdbc.driver:com.mysql.Driver
old.jdbc.url:xxx
old.jdbc.username:xxx
old.jdbc.password:xxx
```
2.写一个CustomerContextHolder,里面的属性ThreadLocal保存的就是当前数据源的key,每一个数据源对应一个key,这里默认是新数据库-new,另一个是老数据库-old

```java
public class CustomerContextHolder {
    public static final String DATA_SOURCE_DEFAULT = "new";
    public static final String DATA_SOURCE_OLD = "old";
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void setCustomerType(String customerType) {
        contextHolder.set(customerType);
    }

    public static String getCustomerType() {
        return contextHolder.get();
    }

    public static void clearCustomerType() {
        contextHolder.remove();
    }
}
```

3.写一个叫DynamicDataSource的类,继承AbstractRoutingDataSource,AbstractRoutingDataSource里面有一个determineCurrentLookupKey方法,是决定
使用哪个数据源key.(相当于说每个数据源有一个key的标识,返回不同的key就是选择了不同的数据源).
```java
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return CustomerContextHolder.getCustomerType();
    }

}
```
4.然后将spring-datasource.xml的配置配好
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-lazy-init="true">

    <!-- DataSource数据 -->
    <bean id="newDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="name" value="wxwwt"/>
        <property name="driverClassName" value="${jdbc.driver}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
        <property name="maxActive" value="20"/>
        <property name="minIdle" value="2"/>
        <property name="initialSize" value="2"/>
        <property name="validationQuery" value="SELECT 1"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>
        <property name="testWhileIdle" value="true"/>
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>
        <property name="minEvictableIdleTimeMillis" value="300000"/>
        <property name="defaultAutoCommit" value="true"/>
        <property name="removeAbandoned" value="true"/>
        <property name="removeAbandonedTimeout" value="60"/>
        <property name="logAbandoned" value="true"/>
        <property name="filters" value="stat"/>
    </bean>

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.wxwwt.web.db.mapper"/>
    </bean>

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="mapperLocations">
            <list>
                <value>classpath*:sqlmap/**/*.xml</value>
            </list>
        </property>
        <property name="dataSource" ref="dynamicDataSource"/>

    </bean>

    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg index="0" ref="sqlSessionFactory"/>
    </bean>

    <bean id="oldDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="name" value="wxwwt"/>
        <property name="driverClassName" value="${old.jdbc.driver}"/>
        <property name="url" value="${old.jdbc.url}"/>
        <property name="username" value="${old.jdbc.username}"/>
        <property name="password" value="${old.jdbc.password}"/>
        <property name="maxActive" value="20"/>
        <property name="minIdle" value="2"/>
        <property name="initialSize" value="2"/>
        <property name="validationQuery" value="SELECT 1"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>
        <property name="testWhileIdle" value="true"/>
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>
        <property name="minEvictableIdleTimeMillis" value="300000"/>
        <property name="defaultAutoCommit" value="true"/>
        <property name="removeAbandoned" value="true"/>
        <property name="removeAbandonedTimeout" value="60"/>
        <property name="logAbandoned" value="true"/>
        <property name="filters" value="stat"/>
    </bean>

    <!-- 动态数据源 -->
    <bean id="dynamicDataSource" class="com.wxwwt.web.config.DynamicDataSource">
        <property name="targetDataSources">
            <map key-type="java.lang.String">
                <entry value-ref="newDataSource" key="new"/>
                <entry value-ref="oldDataSource" key="old"/>
            </map>
        </property>
        <property name="defaultTargetDataSource" ref="newDataSource">
        </property>
    </bean>


    <bean id="txManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dynamicDataSource"/>
    </bean>
</beans>
```
里面配置好数据源newDataSource,oldDataSource.重点是dynamicDataSource里面配置好拥有的数据源,
然后将dynamicDataSource配置到DataSourceTransactionManager里面就大功告成了.

5.使用的使用只需要CustomerContextHolder.setCustomerType(CustomerContextHolder.DATA_SOURCE_OLD);切换到老数据源,因为默认是新数据源.
如果要切换回来就CustomerContextHolder.clearCustomerType().清除掉ThreadLocal中保存的信息.又因为切换数据源的key是在ThreadLocal中的所以是只和线程有关,
一个线程选择老数据源,并不会影响到其他的线程.
