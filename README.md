####  前提：为什么要用持久层框架？

​	这就要说到JDBC存在的问题了，持久层的出现就是为了解决这些问题的。

​	下表为JDBC存在的问题及解决方案：

|            问题             |                    具体体现                     | 解决方案 |
| :-------------------------: | :---------------------------------------------: | :------: |
|       配置信息硬编码        |     Class.forName("com.mysql.jdbc.Driver")      | 配置文件 |
|        频繁获取连接         | 每次请求都调用DriverManager.getConnection(XXXX) | 池化技术 |
| SQL语句，设置参数存在硬编码 |     preparedStatement.setString(1, "xxx");      | 配置文件 |
|       手动封装结果集        |               while（rs.next）{}                |   反射   |

​	了解了JDBC存在的问题，那实现自定义持久层就有了方向，目的就在于解决这些问题。

---



#### 自定义持久层框架

​	实现思路：加载配置文件，根据配置文件生成Configuration配置对象，其中包含了两种信息，一种是数据库连接信息，一种是存放SQL映射的map对象。在进行方法调用时根据其全限定类名和方法名称定为到相应的SQL语句，然后执行JDBC代码

```java
/**
 * 配置信息
 */
public class Configuration {
    //数据源：包含数据库连接的相关信息
    private DataSource dataSource;
    //mapperStatement
    private Map<String,MapperStatement> mapperStatements = new HashMap<>();

   	//getter/setter
}

/**
 * 对应*Mapping.xml文件中的语句
 */
public class MapperStatement {
    //id
    private String id;
    //返回值类型
    private String resultType;
    //参数类型
    private String parameterType;
    //sql语句
    private String sql;
    //sql类型
    private String sqlCommand;
	
    //getter/setter
}

```

---



#### 初识MyBatis

​	主要构件之会话对象，与使用者交互的接口，完成增删改查功能

|         类/接口          |                             作用                             |
| :----------------------: | :----------------------------------------------------------: |
| SqlSessionFactoryBuilder |               根据相关配置生成SqlSessionFatory               |
|     SqlSessionFatory     |                    生成SqlSession会话对象                    |
|        SqlSession        |      发送SQL语句去执行并返回结果；提供获取Mapper的接口       |
|        SQL Mapper        | 提供一个接口，给出SQL语句及映射规则，它负责发送SQl去执行并放回结果 |

​	主要构建之执行器，负责SQL动态语句的生成和查询缓存的维护

|     执行器     |               功能               |
| :------------: | :------------------------------: |
|    Executor    |               接口               |
| BatchExecutor  | 重用语句和批量更新，针对批量专用 |
| ReuseExecutor  |          重用预处理语句          |
| SimpleExecutor |       简易执行器，默认配置       |

核心配置文件SqlConfig.xml，注意每个节点有顺序之分

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <!--加载外部的properties文件-->
    <properties resource="jdbc.properties"></properties>
    
       <!--开启二级缓存  -->
    <settings>
        <setting name="cacheEnabled" value="true"/>
    </settings>

    <!--给实体类的全限定类名给别名-->
    <typeAliases>
        <!--给单独的实体起别名-->
      <!--  <typeAlias type="com.xxx.pojo.User" alias="user"></typeAlias>-->
        <!--批量起别名：该包下所有的类的本身的类名：别名还不区分大小写-->
        <package name="com.xxx.pojo"/>
    </typeAliases>
    
    <!--类型处理器-->
    <typeHandles>
    	<typeHandler></typeHandler>
    </typeHandles>
    
    <!--对象工厂-->
    <ObjectFactory>
    </ObjectFactory>
    
    <!--插件-->
   <plugins>
        <plugin interceptor="com.github.pagehelper.PageHelper">
            <property name="dialect" value="mysql"/>
        </plugin>
        
        <plugin interceptor="tk.mybatis.mapper.mapperhelper.MapperInterceptor">
            <!--指定当前通用mapper接口使用的是哪一个-->
            <property name="mappers" value="tk.mybatis.mapper.common.Mapper"/>
        </plugin>      
    </plugins>

    <!--environments:运行环境-->
    <environments default="development">
        <environment id="development">
            <!--当前事务交由JDBC进行管理-->
            <transactionManager type="JDBC"></transactionManager>
            <!--当前使用mybatis提供的连接池-->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>

    <!--数据库厂商标识-->
    <databaseIdProvider type="DB_VENDOR">
    	 <property name="MySql" value="mysql"/>
    </databaseIdProvider>
    
    <!--引入映射配置文件-->
    <mappers>
        <mapper resource="UserMapper.xml"></mapper>
    </mappers>



</configuration>
```

​	缓存：缓存是存储在内存中的数据，其目的在于提高查询速度且有效降低数据库的压力。MyBatis为我们提供了两种缓存一级缓存和二级缓存。

|   缓存   | 存储结构 |       范围        |                          失效场景                           |
| :------: | :------: | :---------------: | :---------------------------------------------------------: |
| 一级缓存 | map<K,V> |    SqlSession     | 进行增删改并提交事务或调用sqlSession.clearCache()会清空缓存 |
| 二级缓存 | map<K,V> | SqlSessionFactory |                    进行增删改并提交事务                     |

​	插件：插件的初始化是在加载核心配置文件的时候完成的，其被保存在一个list集合中。之后使用责任链模式为四大对象生成代理对象并返回，当代理对象调用方法时就会进入invoke()方法中，在invoke方法中，如果存在签名的拦截方法，这时就会调用插件的intercept()。否，就直接使用反射调用要执行的方法。

​	自定义插件：

​		实现Interceptor接口

​		重写重要的3个方法：

​				intercept(Invocation invocation)：覆盖目标对象原有的方法。invocation对象，通过它可以利用反射调				用目标对象的方法。

​				plugin(Object tartget)：target是被拦截的对象，plugin的作用是为target生成一个代理对象并返回。

​				setProperties(Properties properties)：允许配置插件所需参数。

​		@Intercepts注解声明拦截器，@Signature注册拦截器签名

```java
@Intercepts({
        @Signature(type= StatementHandler.class,
                  method = "prepare",
                  args = {Connection.class,Integer.class})
})
public class MyPlugin implements Interceptor {

    /*
        拦截方法：只要被拦截的目标对象的目标方法被执行时，每次都会执行intercept方法
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("对方法进行了增强....");
        return invocation.proceed(); //原方法执行
    }

    /*
       主要为了把当前的拦截器生成代理存到拦截器链中
     */
    @Override
    public Object plugin(Object target) {
        Object wrap = Plugin.wrap(target, this);
        return wrap;
    }

    /*
        获取配置文件的参数
     */
    @Override
    public void setProperties(Properties properties) {
        System.out.println("获取到的配置文件的参数是："+properties);
    }
}
```

---



#### 设计模式

​	*构建者模式：将一个对象的构建与其表示相分离，使得同样的构建过程可以创建不同的表示

​	动机：在软件系统中，有时候面临着一个复杂对象的创建工作，其通常由各个部分的子对象用一定的算法构成，由于需求变更，这个复杂对象的各个部分经常面临变化，但是将其组合在一起的算法却相对稳定

​	*工厂模式：可以根据不同的参数生产不同实例

​	 动机：在软件系统中，经常面临着创建对象的工作，由于需求的变化，需要创建的对象的具体类型经常变化

​	*代理模式：为指定对象生成一个代理对象，由代理对象控制对目标对象的引用

​	  动机：在面向对象系统中，有些对象由于某种原因（如某些操作需要安全访问），直接访问会给系统访问带来        	不必要的麻烦

