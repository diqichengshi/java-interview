## 1.BeanFactory 和FactoryBean的区别

1.BeanFactory以Factory结尾，**表示它是一个工厂类，用于管理Bean的一个工厂，在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的**。BeanFactory只是个接口，并不是IOC容器的具体实现，但是Spring容器给出了很多种实现，如 DefaultListableBeanFactory、XmlBeanFactory、ApplicationContext等，其中XmlBeanFactory就是常用的一个，该实现将以XML方式描述组成应用的对象及对象间的依赖关系。

2.FactoryBean不是简单的Bean，一般情况下，Spring通过反射机制利用<bean>的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在<bean>中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个org.springframework.bean.factory.FactoryBean的工厂类接口，**用户可以通过实现该接口定制实例化Bean的逻辑**。FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。

## 2.Spring单例

当单例Bean依赖多例Bean时，单例Bean只有一次初始化的机会，它的依赖关系只有在初始化阶段被设置，而它所依赖的多例Bean会不断更新产生新的Bean实例，这将导致单例Bean所依赖的多例Bean得不到更新，每次都得到的是最开始时生成的Bean，这就违背了使用多例的初衷。 
解决该问题有两种解决思路： 
1.放弃依赖注入：主动向容器获取多例，可以实现ApplicationContextAware接口来获取ApplicationContext实例,通过ApplicationContext获取多例对象。 
2.利用方法注入：方法注入是让Spring容器重写Bean中的抽象方法，该方法返回多例，Spring通过CGLIb修改客户端的二进制代码来实现。 

## 3.Spring中的事务管理(记不住)

Spring事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。真正的数据库层的事务提交和回滚是通过binlog或者redo log实现的。

### 3.1.Spring事务的种类：

spring支持编程式事务管理和声明式事务管理两种方式：

1.1.编程式事务管理使用TransactionTemplate。

1.2.声明式事务管理建立在AOP之上的。其本质是通过AOP功能，对方法前后进行拦截，将事务处理的功能编织到拦截的方法中，也就是在目标方法开始之前加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。

声明式事务最大的优点就是不需要在业务逻辑代码中掺杂事务管理的代码，只需在配置文件中做相关的事务规则声明或通过@Transactional注解的方式，便可以将事务规则应用到业务逻辑中。

声明式事务管理要优于编程式事务管理，这正是spring倡导的非侵入式的开发方式，使业务代码不受污染，只要加上注解就可以获得完全的事务支持。唯一不足地方是，最细粒度只能作用到方法级别，无法做到像编程式事务那样可以作用到代码块级别。

### 3.2.spring的事务传播行为：

spring事务的传播行为说的是，当多个事务同时存在的时候，spring如何处理这些事务的行为。

2.1.PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置是最常用的设置。

2.2.PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行。‘

2.3. PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在事务，就抛出异常。

2.4. PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。

2.5.PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

2.6. PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。

2.7. PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则按REQUIRED属性执行。

### 3.3.Spring中的隔离级别：

3.1.ISOLATION_DEFAULT：这是个 PlatfromTransactionManager 默认的隔离级别，使用数据库默认的事务隔离级别。

3.2. ISOLATION_READ_UNCOMMITTED：读未提交，允许另外一个事务可以看到这个事务未提交的数据。

3.3.ISOLATION_READ_COMMITTED：读已提交，保证一个事务修改的数据提交后才能被另一事务读取，而且能看到该事务对已有记录的更新。

3.4. ISOLATION_REPEATABLE_READ：可重复读，保证一个事务修改的数据提交后才能被另一事务读取，但是不能看到该事务对已有记录的更新。

3.5. ISOLATION_SERIALIZABLE：一个事务在执行的过程中完全看不到其他事务对数据库所做的更新。

## 4.Spring AOP的实现机制(动态代理)

静态代理是编译阶段生成AOP代理类，也就是说生成的字节码就织入了增强后的AOP对象;动态代理则不会修改字节码，而是在内存中临时生成一个AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

Spring AOP中的动态代理主要有两种方式，[JDK动态代理](http://mp.weixin.qq.com/s?__biz=MzU5NTAzNjM0Mw==&mid=2247486158&idx=2&sn=ffa8e107159fcb47a44f090abc7ef7d3&chksm=fe795b16c90ed200340b810ccd89c7189b60915e8bef9ea2cce230a8b76afbc60bea1d2f6e67&scene=21#wechat_redirect)和CGLIB动态代理.JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口.JDK动态代理的核心是InvocationHandler接口和代理类。

如果**目标类没有实现接口，那么Spring AOP会选择使用CGLIB来动态代理目标类**，是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为最终的，那么它是无法使用CGLIB做动态代理的，诸如私人的方法也是不可以作为切面的。

