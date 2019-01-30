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


