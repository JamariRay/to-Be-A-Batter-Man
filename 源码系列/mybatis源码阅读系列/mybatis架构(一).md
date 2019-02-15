# <center><font color ="f0a1a8"> 《架构之路--mybatis篇（一）》</font></center>
## **mybatis官方文档阅读**
想学任何一门技术，最好的方式就是阅读官方文档，可能有些同学英语水平较弱（ps:本人就是一枚英语渣），不太爱看官方文档，庆幸[mybatis官方有中文文档](http://www.mybatis.org/mybatis-3/zh/getting-started.html)，那我们就来看看吧！

*  1、项目初始化</br>
>每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为中心的。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先定制的 Configuration 的实例构建出 SqlSessionFactory 的实例。

我们来解读一下，每个mybatis 的实例都是以SqlSessionFactory的实例为中心的，我们知道mybatis是对原生JDBC进行了封装，那我们先假装没用过mybatis，通过字面意思大胆的猜测下SqlSessionFactory的作用是，获取sql连接，执行sql语句，然后释放资源
```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
首先去获取mybatis-config.xml（自定义mybatis配置文件），用Resources加载字节流，再用SqlSessionFactoryBuilder来创建SqlSessionFactory(ps:常见的builder模式)
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
} finally {
  session.close();
}
```
首先我们看到sqlSessionFactory只帮我们开启了线程，并没有关闭线程，好像sqlSessionFactory并没有帮我们关闭连接，好了这个问题先留在这里，我们继续阅读文档:


>* 依赖注入框架可以创建线程安全的、基于事务的 SqlSession 和映射器（mapper）并将它们直接注入到你的 bean 中，因此可以直接忽略它们的生命周期。如果对如何通过依赖注入框架来使用 MyBatis 感兴趣可以研究一下 MyBatis-Spring 或 MyBatis-Guice 两个子项目。
>* **SqlSessionFactoryBuilder**</br>
这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但是最好还是不要让其一直存在以保证所有的 XML 解析资源开放给更重要的事情。
>* **SqlSessionFactory**</br>
SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由对它进行清除或重建。使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏味道（bad smell）”。因此 SqlSessionFactory 的最佳作用域是应用作用域。有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。
>* **SqlSession**</br>
每个线程都应该有它自己的 SqlSession 实例。SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。也绝不能将 SqlSession 实例的引用放在任何类型的管理作用域中，比如 Servlet 架构中的 HttpSession。如果你现在正在使用一种 Web 框架，要考虑 SqlSession 放在一个和 HTTP 请求对象相似的作用域中。换句话说，每次收到的 HTTP 请求，就可以打开一个 SqlSession，返回一个响应，就关闭它。这个关闭操作是很重要的，你应该把这个关闭操作放到 finally 块中以确保每次都能执行关闭。下面的示例就是一个确保 SqlSession 关闭的标准模式：

```java
SqlSession session = sqlSessionFactory.openSession();
try {
  // do work
} finally {
  session.close();
}
```
读到这么我们也就知道了SqlSessionFactory并没有帮我们关闭session，那么平时我们整合Mybatis、Spring的时候，Session都是关闭了的，也就是说Spring整合mybatis后，Spring帮我们关闭了Session。这里我们又留一个猜想，这个问题我们留在Spring框架中解决。根据作用域相关引文我们知道核心的类是SqlSessionFactory和SqlSession，我们开始调试Mybatis源码，针对SqlSessionFactory和SqlSession这两个类来看看mybatis在执行过程中做了什么

## 源码调试
我们首先在编写一段测试案例
**实体类**
```java
package com.bestlxh.mybatis.learn;

public class User {
    private Integer id;
    private String name;
    public void setId(Integer id){
        this.id = id;
    }
    public void setName(String name){
        this.name =name;
    }
}
```
**mybatis-config.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <!-- 配置数据库连接信息 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://localhost:3306/learn" />
                <property name="username" value="root" />
                <property name="password" value="BESTLXH!" />
            </dataSource>
        </environment>
    </environments>
<mappers>
    <mapper resource="UserMapper.xml"/>
</mappers>
</configuration>
```
**userMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//ibatis.apache.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.bestlxh.mybatis.learn.UserMapper">

    <insert id="addUser">
		insert into user(id,name) values(#{id},#{name})
	</insert>

</mapper>

```
测试类：**UserTest**
```java
package com.bestlxh.mybatis.learn;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

public class UserTest {

    public static void main(String[] args) {
        User user = new User();
        user.setId(2);
        user.setName("xiaohei");
        String resource = "mybatis-configure.xml";
        try {
            InputStream inputStream = Resources.getResourceAsStream(resource);
            SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
            userMapper.addUser(user);
            sqlSession.commit();
            sqlSession.close();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}

```

`InputStream inputStream = Resources.getResourceAsStream(resource);`处打上断点，debug跑起来。整个调试过程比较枯燥，而且代码也较多，这里就不一一展示，到时候我会录制调试视频，然后上传的bilibili上面，通过iframe的方式内嵌到文本中，好了我们先来讲几个重要的类
## mybatis 获取配置文件的流程
1. **Resource** mybatis 提供的工具类用于加载配置文件，得到一个输出流。调试的时候我们发现返回的输出流是一份 BufferedInputStream
2. 但返回了 InputStream 流之后调取 **SqlSessionFactoryBuilder** 的 builder 方法实际上会调取他的重构方法我们来看代码：
```java  
//首先调取的方法   
public SqlSessionFactory build(InputStream inputStream) {
    //调取重载的方法
        return this.build((InputStream)inputStream, (String)null, (Properties)null);
    }

    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        SqlSessionFactory var5;
        try {
            //这里要注意 XMLConfigBuilder 很重要 是
            XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
            //根据单词parse 我们大致可以猜测这是对xml进行了解析,而且冲下面的build()方法 中我们看到传的是一个Configuration对象，联系到mybatis-config.xml的根标签，我们也可以大致得到信息
            var5 = this.build(parser.parse());
        } catch (Exception var14) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", var14);
        } finally {
            ErrorContext.instance().reset();

            try {
                inputStream.close();
            } catch (IOException var13) {
            }

        }

        return var5;
    }
   
    public SqlSessionFactory build(Configuration config) {
        //返回的是一个sqlSessionFactroy的一个实例
        return new DefaultSqlSessionFactory(config);
    }
}
```
在代码调试的过程中，笔者发现mybatis的作者，很喜欢用重载的方法，看源码的时候你会看到大量调取重载的实例，读者你们觉得这样的好处是什么呢？

3.在调取 `XMLConfigBuilder(InputStream inputStream, String environment, Properties props)` 构造方法时会   XPathParser 这个类的构造方法，这个就是对 mybatis-config 进行解析的类：
```java
 public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    //XMLMapperEntityResolver这个类主要是用于解析DTD Mybatis 主要用于区分config DTD和Mapper DTD ，细心的网友应该有注意到 mapper文件 和 config文件dtd的类型不同
    //XPathParser这个就是对配置文件解析的核心类了
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
}
```
我们再来看看代码执行到这里是 parser.parse()方法：
```java
public Configuration parse() {
    //sqlSessionFactroyBuilder 只能创建一次
    if (this.parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    } else {
        this.parsed = true;
        //解析</configuration>标签
        this.parseConfiguration(this.parser.evalNode("/configuration"));
        return this.configuration;
        }
    }
private void parseConfiguration(XNode root) {
    /**
     *这里对配置文件中标签都进行了解析 
     *
     */
    try {
        this.propertiesElement(root.evalNode("properties"));
        Properties settings = this.settingsAsProperties(root.evalNode("settings"));
        this.loadCustomVfs(settings);
        this.typeAliasesElement(root.evalNode("typeAliases"));
        this.pluginElement(root.evalNode("plugins"));
        this.objectFactoryElement(root.evalNode("objectFactory"));
        this.objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        this.reflectorFactoryElement(root.evalNode("reflectorFactory"));
        this.settingsElement(settings);
        this.environmentsElement(root.evalNode("environments"));
        this.databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        this.typeHandlerElement(root.evalNode("typeHandlers"));
        this.mapperElement(root.evalNode("mappers"));
        } catch (Exception var3) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + var3, var3);
        }
    }
```
上面我们看到 `this.parseConfiguration` 方法中调用了一个 this.parser.evalNode("/configuration") 实际上调用了他的重载方法:
```java
//解析节点调用的方法 传入节点和节点的表达式
public XNode evalNode(Object root, String expression) {
        //Document 节点 XML 解析分为DOM解析SAX解析 mybatis财通SAX解析
        Node node = (Node)this.evaluate(expression, root, XPathConstants.NODE);
        return node == null ? null : new XNode(this, node, this.variables);
    }
```
这里的 Configuration 类个人觉得很重要，大家可以自己去看一下。好了通过上面的流程，就基本上拿到了所有的配置信息了，我们发现返回的是一个 DefaultSqlSessionFactory ,并且讲Configuration信息保存到这个类里面，也就是说调用 openSession() 方法，实际上就是调用 DefaultSqlSessionFactory 这个类的 openSession() 方法 。程序执行下一步 我们去看看 DefaultSqlSessionFactory. openSession() 方法。
```java
  public SqlSession openSession() {
    //方法嵌套调用 ExecutorType 是个枚举类 包含SIMPLE REUSE BATCH; 返回的是SIMPLE 
    return this.openSessionFromDataSource(this.configuration.getDefaultExecutorType(), (TransactionIsolationLevel)null, false);
    }
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;

    DefaultSqlSession var8;
    try {
        //这里就是拿到配置信息
        Environment environment = this.configuration.getEnvironment();
        TransactionFactory transactionFactory = this.getTransactionFactoryFromEnvironment(environment);
        //注册事务
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        //执行器对象 返回的是一个SimpleExecutor对象 然后开启缓存讲Executor交给CachingExecutor
        Executor executor = this.configuration.newExecutor(tx, execType);
        var8 = new DefaultSqlSession(this.configuration, executor, autoCommit);
        } catch (Exception var12) {
            this.closeTransaction(tx);
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + var12, var12);
        } finally {
            ErrorContext.instance().reset();
        }

        return var8;
    }
```
我们来看看SimpleExecutor这个对象：
```java
/**
 * sql 执行对象
 */
public class SimpleExecutor extends BaseExecutor {
    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        //调父类BaseExecutor的构造方法
        super(configuration, transaction);
    }

    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;

        int var6;
        try {
            Configuration configuration = ms.getConfiguration();
            //statement的处理类
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, (ResultHandler)null, (BoundSql)null);
            stmt = this.prepareStatement(handler, ms.getStatementLog());
            var6 = handler.update(stmt);
        } finally {
            //关闭资源
            this.closeStatement(stmt);
        }

        return var6;
    }

    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;

        List var9;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this.wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
            stmt = this.prepareStatement(handler, ms.getStatementLog());
            var9 = handler.query(stmt, resultHandler);
        } finally {
            this.closeStatement(stmt);
        }

        return var9;
    }
    //查询游标
    protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(this.wrapper, ms, parameter, rowBounds, (ResultHandler)null, boundSql);
        Statement stmt = this.prepareStatement(handler, ms.getStatementLog());
        return handler.queryCursor(stmt);
    }

    public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        return Collections.emptyList();
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Connection connection = this.getConnection(statementLog);
        Statement stmt = handler.prepare(connection, this.transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }
}

```
上面我们看到SimpleExecutor调用了父类BaseExecutor的有参构造，我们跟进去看看
```java
//这个类也相当重要
 protected BaseExecutor(Configuration configuration, Transaction transaction) {
        this.transaction = transaction;
        this.deferredLoads = new ConcurrentLinkedQueue();
        //缓存 内部维护了一个Hashmap
        this.localCache = new PerpetualCache("LocalCache");
        this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
        this.closed = false;
        this.configuration = configuration;
        this.wrapper = this;
    }
```
好了基本上这么一圈走下来 SqlSession就生成了

