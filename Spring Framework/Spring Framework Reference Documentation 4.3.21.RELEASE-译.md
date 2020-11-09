## 17.5 Declarative transaction management

>  Most Spring Framework users choose declarative transaction management. This option has the least impact on application code, and hence is most consistent with the ideals of a *non-invasive* lightweight container.

The Spring Framework’s declarative transaction management is made possible with Spring aspect-oriented programming (AOP), although, as the transactional aspects code comes with the Spring Framework distribution and may be used in a boilerplate fashion, AOP concepts do not generally have to be understood to make effective use of this code.

The Spring Framework’s declarative transaction management is similar to EJB CMT in that you can specify transaction behavior (or lack of it) down to individual method level. It is possible to make a <font color = red>`setRollbackOnly()`</font> call within a transaction context if necessary. The differences between the two types of transaction management are:

- Unlike EJB CMT, which is tied to JTA, the Spring Framework’s declarative transaction management works in any environment. It can work with JTA transactions or local transactions using JDBC, JPA, Hibernate or JDO by simply adjusting the configuration files.
- You can apply the Spring Framework declarative transaction management to any class, not merely special classes such as EJBs.
- The Spring Framework offers declarative [*rollback rules*,](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#transaction-declarative-rolling-back)a feature with no EJB equivalent. Both programmatic and declarative support for rollback rules is provided.
- The Spring Framework enables you to customize transactional behavior, by using AOP. For example, you can insert custom behavior in the case of transaction rollback. You can also add arbitrary advice, along with the transactional advice. With EJB CMT, you cannot influence the container’s transaction management except with <font color = red>`setRollbackOnly()`</font>.
- The Spring Framework does not support propagation of transaction contexts across remote calls, as do high-end application servers. If you need this feature, we recommend that you use EJB. However, consider carefully before using such a feature, because normally, one does not want transactions to span remote calls.

> <font color = red>**Where is TransactionProxyFactoryBean?**</font>
>
> Declarative transaction configuration in versions of Spring 2.0 and above differs considerably from previous versions of Spring. The main difference is that there is no longer any need to configure <font color = red>`TransactionProxyFactoryBean`</font> beans.
>
> The pre-Spring 2.0 configuration style is still 100% valid configuration; think of the new <font color = red>`<tx:tags/>`</font> as simply defining <font color = red>`TransactionProxyFactoryBean`</font> beans on your behalf.

The concept of rollback rules is important: they enable you to specify which exceptions (and throwables) should cause automatic rollback. You specify this declaratively, in configuration, not in Java code. So, although you can still call <font color = red>`setRollbackOnly()`</font> on the <font color = red>`TransactionStatus`</font> object to roll back the current transaction back, most often you can specify a rule that <font color = red>`MyApplicationException`</font> must always result in rollback. The significant advantage to this option is that business objects do not depend on the transaction infrastructure. For example, they typically do not need to import Spring transaction APIs or other Spring APIs.

Although EJB container default behavior automatically rolls back the transaction on a *system exception* (usually a runtime exception), EJB CMT does not roll back the transaction automatically on an*application exception* (that is, a checked exception other than <font color = red>`java.rmi.RemoteException`</font>). While the Spring default behavior for declarative transaction management follows EJB convention (roll back is automatic only on unchecked exceptions), it is often useful to customize this behavior.

### 17.5.1 Understanding the Spring Framework’s declarative transaction implementation

It is not sufficient to tell you simply to annotate your classes with the <font color = red>`@Transactional`</font> annotation, add <font color = red>`@EnableTransactionManagement`</font> to your configuration, and then expect you to understand how it all works. This section explains the inner workings of the Spring Framework’s declarative transaction infrastructure in the event of transaction-related issues.

The most important concepts to grasp with regard to the Spring Framework’s declarative transaction support are that this support is enabled [*via AOP proxies*](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop-understanding-aop-proxies), and that the transactional advice is driven by *metadata* (currently XML- or annotation-based). The combination of AOP with transactional metadata yields an AOP proxy that uses a <font color = red>`TransactionInterceptor`</font> in conjunction with an appropriate <font color = red>`PlatformTransactionManager`</font> implementation to drive transactions *around method invocations*.

> Spring AOP is covered in [Chapter 11, *Aspect Oriented Programming with Spring*](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop).

Conceptually, calling a method on a transactional proxy looks like this…

![tx](markdown/17.5 Declarative transaction management.assets/tx.png)

### 17.5.2 Example of declarative transaction implementation

Consider the following interface, and its attendant implementation. This example uses <font color = red>`Foo`</font> and <font color = red>`Bar`</font> classes as placeholders so that you can concentrate on the transaction usage without focusing on a particular domain model. For the purposes of this example, the fact that the <font color = red>`DefaultFooService`</font> class throws <font color = red>`UnsupportedOperationException`</font> instances in the body of each implemented method is good; it allows you to see transactions created and then rolled back in response to the <font color = red>`UnsupportedOperationException`</font> instance.

```java
// the service interface that we want to make transactional

package x.y.service;

public interface FooService {

    Foo getFoo(String fooName);

    Foo getFoo(String fooName, String barName);

    void insertFoo(Foo foo);

    void updateFoo(Foo foo);

}
```

```java
// an implementation of the above interface

package x.y.service;

public class DefaultFooService implements FooService {

    public Foo getFoo(String fooName) {
        throw new UnsupportedOperationException();
    }

    public Foo getFoo(String fooName, String barName) {
        throw new UnsupportedOperationException();
    }

    public void insertFoo(Foo foo) {
        throw new UnsupportedOperationException();
    }

    public void updateFoo(Foo foo) {
        throw new UnsupportedOperationException();
    }

}
```

Assume that the first two methods of the <font color = red>`FooService`</font> interface, <font color = red>`getFoo(String)`</font> and <font color = red>`getFoo(String, String)`</font>, must execute in the context of a transaction with read-only semantics, and that the other methods, <font color = red>`insertFoo(Foo)`</font> and <font color = red>`updateFoo(Foo)`</font>, must execute in the context of a transaction with read-write semantics. The following configuration is explained in detail in the next few paragraphs.

```xml
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- the transactional advice (what 'happens'; see the <aop:advisor/> bean below) -->
    <tx:advice id="txAdvice" transaction-manager="txManager">
        <!-- the transactional semantics... -->
        <tx:attributes>
            <!-- all methods starting with 'get' are read-only -->
            <tx:method name="get*" read-only="true"/>
            <!-- other methods use the default transaction settings (see below) -->
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- ensure that the above transactional advice runs for any execution
        of an operation defined by the FooService interface -->
    <aop:config>
        <aop:pointcut id="fooServiceOperation" expression="execution(* x.y.service.FooService.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceOperation"/>
    </aop:config>

    <!-- don't forget the DataSource -->
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"/>
        <property name="url" value="jdbc:oracle:thin:@rj-t42:1521:elvis"/>
        <property name="username" value="scott"/>
        <property name="password" value="tiger"/>
    </bean>

    <!-- similarly, don't forget the PlatformTransactionManager -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>
```

Examine the preceding configuration. You want to make a service object, the <font color = red>`fooService`</font> bean, transactional. The transaction semantics to apply are encapsulated in the <font color = red>`<tx:advice/>`</font> definition. The <font color = red>`<tx:advice/>`</font> definition reads as "*… all methods on starting with <font color = red>`'get'`</font> are to execute in the context of a read-only transaction, and all other methods are to execute with the default transaction semantics*". The <font color = red>`transaction-manager`</font> attribute of the <font color = red>`<tx:advice/>`</font> tag is set to the name of the<font color = red>`PlatformTransactionManager`</font> bean that is going to *drive* the transactions, in this case, the <font color = red>`txManager`</font> bean.

> You can omit the `transaction-manager` attribute in the transactional advice (`<tx:advice/>`) if the bean name of the `PlatformTransactionManager`that you want to wire in has the name `transactionManager`. If the `PlatformTransactionManager` bean that you want to wire in has any other name, then you must use the `transaction-manager` attribute explicitly, as in the preceding example.

The `<aop:config/>` definition ensures that the transactional advice defined by the `txAdvice` bean executes at the appropriate points in the program. First you define a pointcut that matches the execution of any operation defined in the `FooService` interface ( `fooServiceOperation`). Then you associate the pointcut with the `txAdvice` using an advisor. The result indicates that at the execution of a `fooServiceOperation`, the advice defined by `txAdvice` will be run.

The expression defined within the `<aop:pointcut/>` element is an AspectJ pointcut expression; see [Chapter 11, *Aspect Oriented Programming with Spring*](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop) for more details on pointcut expressions in Spring.

A common requirement is to make an entire service layer transactional. The best way to do this is simply to change the pointcut expression to match any operation in your service layer. For example:

```xml
<aop:config>
    <aop:pointcut id="fooServiceMethods" expression="execution(* x.y.service.*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="fooServiceMethods"/>
</aop:config>
```

> *In this example it is assumed that all your service interfaces are defined in the `x.y.service` package; see [Chapter 11, \*Aspect Oriented Programming with Spring\*](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop) for more details.*

Now that we’ve analyzed the configuration, you may be asking yourself, "*Okay… but what does all this configuration actually do?*".

The above configuration will be used to create a transactional proxy around the object that is created from the `fooService` bean definition. The proxy will be configured with the transactional advice, so that when an appropriate method is invoked *on the proxy*, a transaction is started, suspended, marked as read-only, and so on, depending on the transaction configuration associated with that method. Consider the following program that test drives the above configuration:

```java
public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("context.xml", Boot.class);
        FooService fooService = (FooService) ctx.getBean("fooService");
        fooService.insertFoo (new Foo());
    }
}
```

The output from running the preceding program will resemble the following. (The Log4J output and the stack trace from the UnsupportedOperationException thrown by the insertFoo(..) method of the DefaultFooService class have been truncated for clarity.)

```
<!-- the Spring container is starting up... -->
[AspectJInvocationContextExposingAdvisorAutoProxyCreator] - Creating implicit proxy for bean 'fooService' with 0 common interceptors and 1 specific interceptors

<!-- the DefaultFooService is actually proxied -->
[JdkDynamicAopProxy] - Creating JDK dynamic proxy for [x.y.service.DefaultFooService]

<!-- ... the insertFoo(..) method is now being invoked on the proxy -->
[TransactionInterceptor] - Getting transaction for x.y.service.FooService.insertFoo

<!-- the transactional advice kicks in here... -->
[DataSourceTransactionManager] - Creating new transaction with name [x.y.service.FooService.insertFoo]
[DataSourceTransactionManager] - Acquired Connection [org.apache.commons.dbcp.PoolableConnection@a53de4] for JDBC transaction

<!-- the insertFoo(..) method from DefaultFooService throws an exception... -->
[RuleBasedTransactionAttribute] - Applying rules to determine whether transaction should rollback on java.lang.UnsupportedOperationException
[TransactionInterceptor] - Invoking rollback for transaction on x.y.service.FooService.insertFoo due to throwable [java.lang.UnsupportedOperationException]

<!-- and the transaction is rolled back (by default, RuntimeException instances cause rollback) -->
[DataSourceTransactionManager] - Rolling back JDBC transaction on Connection [org.apache.commons.dbcp.PoolableConnection@a53de4]
[DataSourceTransactionManager] - Releasing JDBC Connection after transaction
[DataSourceUtils] - Returning JDBC Connection to DataSource

Exception in thread "main" java.lang.UnsupportedOperationException at x.y.service.DefaultFooService.insertFoo(DefaultFooService.java:14)
<!-- AOP infrastructure stack trace elements removed for clarity -->
at $Proxy0.insertFoo(Unknown Source)
at Boot.main(Boot.java:11)
```

### 17.5.3 Rolling back a declarative transaction

The previous section outlined the basics of how to specify transactional settings for classes, typically service layer classes, declaratively in your application. This section describes how you can control the rollback of transactions in a simple declarative fashion.

The recommended way to indicate to the Spring Framework’s transaction infrastructure that a transaction’s work is to be rolled back is to throw an <font color = red>`Exception`</font> from code that is currently executing in the context of a transaction. The Spring Framework’s transaction infrastructure code will catch any unhandled <font color = red>`Exception`</font> as it bubbles up the call stack, and make a determination whether to mark the transaction for rollback.

In its default configuration, the Spring Framework’s transaction infrastructure code *only* marks a transaction for rollback in the case of runtime, unchecked exceptions; that is, when the thrown exception is an instance or subclass of <font color = red>`RuntimeException`</font>. ( <font color = red>`Error`</font>s will also - by default - result in a rollback). Checked exceptions that are thrown from a transactional method do *not* result in rollback in the default configuration.

You can configure exactly which <font color = red>`Exception`</font> types mark a transaction for rollback, including checked exceptions. The following XML snippet demonstrates how you configure rollback for a checked, application-specific <font color = red>`Exception`</font> type.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
    <tx:method name="get*" read-only="true" rollback-for="NoProductInStockException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

You can also specify 'no rollback rules', if you do *not* want a transaction rolled back when an exception is thrown. The following example tells the Spring Framework’s transaction infrastructure to commit the attendant transaction even in the face of an unhandled `InstrumentNotFoundException`.

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="updateStock" no-rollback-for="InstrumentNotFoundException"/>
    <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

When the Spring Framework’s transaction infrastructure catches an exception and it consults configured rollback rules to determine whether to mark the transaction for rollback, the *strongest* matching rule wins. So in the case of the following configuration, any exception other than an `InstrumentNotFoundException` results in a rollback of the attendant transaction.

```xml
<tx:advice id="txAdvice">
    <tx:attributes>
    <tx:method name="*" rollback-for="Throwable" no-rollback-for="InstrumentNotFoundException"/>
    </tx:attributes>
</tx:advice>
```

You can also indicate a required rollback *programmatically*. Although very simple, this process is quite invasive, and tightly couples your code to the Spring Framework’s transaction infrastructure:

```java
public void resolvePosition() {
    try {
        // some business logic...
    } catch (NoProductInStockException ex) {
        // trigger rollback programmatically
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

You are strongly encouraged to use the declarative approach to rollback if at all possible. Programmatic rollback is available should you absolutely need it, but its usage flies in the face of achieving a clean POJO-based architecture.

### 17.5.4 Configuring different transactional semantics for different beans

Consider the scenario where you have a number of service layer objects, and you want to apply a *totally different* transactional configuration to each of them. You do this by defining distinct `<aop:advisor/>` elements with differing `pointcut` and `advice-ref` attribute values.

As a point of comparison, first assume that all of your service layer classes are defined in a root `x.y.service` package. To make all beans that are instances of classes defined in that package (or in subpackages) and that have names ending in <font color = red>`Service`</font> have the default transactional configuration, you would write the following:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="serviceOperation"
                expression="execution(* x.y.service..*Service.*(..))"/>

        <aop:advisor pointcut-ref="serviceOperation" advice-ref="txAdvice"/>

    </aop:config>

    <!-- these two beans will be transactional... -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>
    <bean id="barService" class="x.y.service.extras.SimpleBarService"/>

    <!-- ... and these two beans won't -->
    <bean id="anotherService" class="org.xyz.SomeService"/> <!-- (not in the right package) -->
    <bean id="barManager" class="x.y.service.SimpleBarManager"/> <!-- (doesn't end in 'Service') -->

    <tx:advice id="txAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a PlatformTransactionManager omitted... -->

</beans>
```

The following example shows how to configure two distinct beans with totally different transactional settings.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="defaultServiceOperation"
                expression="execution(* x.y.service.*Service.*(..))"/>

        <aop:pointcut id="noTxServiceOperation"
                expression="execution(* x.y.service.ddl.DefaultDdlManager.*(..))"/>

        <aop:advisor pointcut-ref="defaultServiceOperation" advice-ref="defaultTxAdvice"/>

        <aop:advisor pointcut-ref="noTxServiceOperation" advice-ref="noTxAdvice"/>

    </aop:config>

    <!-- this bean will be transactional (see the 'defaultServiceOperation' pointcut) -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- this bean will also be transactional, but with totally different transactional settings -->
    <bean id="anotherFooService" class="x.y.service.ddl.DefaultDdlManager"/>

    <tx:advice id="defaultTxAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <tx:advice id="noTxAdvice">
        <tx:attributes>
            <tx:method name="*" propagation="NEVER"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a PlatformTransactionManager omitted... -->

</beans>
```

### 17.5.5 \<tx:advice/> settings

This section summarizes the various transactional settings that can be specified using the `<tx:advice/>` tag. The default `<tx:advice/>` settings are:

- [Propagation setting](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#tx-propagation) is `REQUIRED.`
- Isolation level is `DEFAULT.`
- Transaction is read/write.
- Transaction timeout defaults to the default timeout of the underlying transaction system, or none if timeouts are not supported.
- Any `RuntimeException` triggers rollback, and any checked `Exception` does not.

You can change these default settings; the various attributes of the `<tx:method/>` tags that are nested within `<tx:advice/>` and `<tx:attributes/>` tags are summarized below:

| Attribute         | Required? | Default  | Description                                                  |
| ----------------- | --------- | -------- | ------------------------------------------------------------ |
| `name`            | Yes       |          | Method name(s) with which the transaction attributes are to be associated. The wildcard (*) character can be used to associate the same transaction attribute settings with a number of methods; for example, `get*`, `handle*`, `on*Event`, and so forth. |
| `propagation`     | No        | REQUIRED | Transaction propagation behavior.                            |
| `isolation`       | No        | DEFAULT  | Transaction isolation level. Only applicable to propagation REQUIRED or REQUIRES_NEW. |
| `timeout`         | No        | -1       | Transaction timeout (seconds). Only applicable to propagation REQUIRED or REQUIRES_NEW. |
| `read-only`       | No        | false    | Read/write vs. read-only transaction. Only applicable to REQUIRED or REQUIRES_NEW. |
| `rollback-for`    | No        |          | `Exception(s)` that trigger rollback; comma-delimited. For example,`com.foo.MyBusinessException,ServletException.` |
| `no-rollback-for` | No        |          | `Exception(s)` that do *not* trigger rollback; comma-delimited. For example,`com.foo.MyBusinessException,ServletException.` |

### 17.5.6 Using @Transactional

In addition to the XML-based declarative approach to transaction configuration, you can use an annotation-based approach. Declaring transaction semantics directly in the Java source code puts the declarations much closer to the affected code. There is not much danger of undue coupling, because code that is meant to be used transactionally is almost always deployed that way anyway.

> The standard <font color = red>`javax.transaction.Transactional`</font> annotation is also supported as a drop-in replacement to Spring’s own annotation. Please refer to JTA 1.2 documentation for more details.

The ease-of-use afforded by the use of the <font color = red>`@Transactional`</font> annotation is best illustrated with an example, which is explained in the text that follows. Consider the following class definition:

```java
// the service class that we want to make transactional
@Transactional
public class DefaultFooService implements FooService {

    Foo getFoo(String fooName);

    Foo getFoo(String fooName, String barName);

    void insertFoo(Foo foo);

    void updateFoo(Foo foo);
}
```

Used at the class level as above, the annotation indicates a default for all methods of the declaring class (as well as its subclasses). Alternatively, each method can get annotated individually. Note that a class-level annotation does not apply to ancestor classes up the class hierarchy; in such a scenario, methods need to be locally redeclared in order to participate in a subclass-level annotation.

When the above POJO is defined as a bean in a Spring IoC container, the bean instance can be made transactional by adding merely *one* line of XML configuration:

```xml
<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- enable the configuration of transactional behavior based on annotations -->
    <tx:annotation-driven transaction-manager="txManager"/><!-- a PlatformTransactionManager is still required -->

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- (this dependency is defined somewhere else) -->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>
```

> You can omit the <font color = red>`transaction-manager`</font> attribute in the <font color = red>`<tx:annotation-driven/>`</font> tag if the bean name of the <font color = red>`PlatformTransactionManager`</font> that you want to wire in has the name <font color = red>`transactionManager`</font>. If the `PlatformTransactionManager` bean that you want to dependency-inject has any other name, then you have to use the <font color = red>`transaction-manager`</font> attribute explicitly, as in the preceding example.

> The <font color = red>`@EnableTransactionManagement`</font> annotation provides equivalent support if you are using Java based configuration. Simply add the annotation to a <font color = red>`@Configuration`</font> class. See the javadocs for full details.

> <font color = red>**Method visibility and @Transactional**</font>
>
> When using proxies, you should apply the <font color = red>`@Transactional`</font> annotation only to methods with *public* visibility. If you do annotate protected, private or package-visible methods with the <font color = red>`@Transactional`</font> annotation, no error is raised, but the annotated method does not exhibit the configured transactional settings. Consider the use of AspectJ (see below) if you need to annotate non-public methods.

the use of AspectJ (see below) if you need to annotate non-public methods.

You can place the <font color = red>`@Transactional`</font> annotation before an interface definition, a method on an interface, a class definition, or a *public* method on a class. However, the mere presence of the <font color = red>`@Transactional`</font> annotation is not enough to activate the transactional behavior. The <font color = red>`@Transactional`</font> annotation is simply metadata that can be consumed by some runtime infrastructure that is <font color = red>`@Transactional`</font>-aware and that can use the metadata to configure the appropriate beans with transactional behavior. In the preceding example, the <font color = red>`<tx:annotation-driven/>`</font> element *switches on* the transactional behavior.

>  Spring recommends that you only annotate concrete classes (and methods of concrete classes) with the <font color = red>`@Transactional`</font> annotation, as opposed to annotating interfaces. You certainly can place the<font color = red> `@Transactional`</font> annotation on an interface (or an interface method), but this works only as you would expect it to if you are using interface-based proxies. The fact that Java annotations are *not inherited from interfaces* means that if you are using class-based proxies ( <font color = red>`proxy-target-class="true"`</font>) or the weaving-based aspect ( <font color = red>`mode="aspectj"`</font>), then the transaction settings are not recognized by the proxying and weaving infrastructure, and the object will not be wrapped in a transactional proxy, which would be decidedly *bad*.

> In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted. This means that self-invocation, in effect, a method within the target object calling another method of the target object, will not lead to an actual transaction at runtime even if the invoked method is marked with <font color = red>`@Transactional`</font>. Also, the proxy must be fully initialized to provide the expected behaviour so you should not rely on this feature in your initialization code, i.e. <font color = red>`@PostConstruct`</font>.

Consider the use of AspectJ mode (see mode attribute in table below) if you expect self-invocations to be wrapped with transactions as well. In this case, there will not be a proxy in the first place; instead, the target class will be weaved (that is, its byte code will be modified) in order to turn <font color = red>`@Transactional`</font> into runtime behavior on any kind of method.

> **Annotation driven transaction settings:**
>
> > **XML Attribute:** transaction-manager
> >
> > **Annotation Attribute:** N/A(See TransactionManagementConfigurer javadoc)
> >
> > **Default:** transactionManager
> >
> > **Description:** Name of transaction manager to use. Only required if the name of the transaction manager is not <font color = red>`transactionManager`</font>, as in the example above.
>
> > **XML Attribute:** mode
> >
> > **Annotation Attribute:** mode
> >
> > **Default:** proxy
> >
> > **Description:** The default mode "proxy" processes annotated beans to be proxied using Spring’s AOP framework (following proxy semantics, as discussed above, applying to method calls coming in through the proxy only). The alternative mode "aspectj" instead weaves the affected classes with Spring’s AspectJ transaction aspect, modifying the target class byte code to apply to any kind of method call. AspectJ weaving requires spring-aspects.jar in the classpath as well as load-time weaving (or compile-time weaving) enabled. (See [the section called “Spring configuration”](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop-aj-ltw-spring) for details on how to set up load-time weaving.)
>
> > **XML Attribute:** proxy-target-class
> >
> > **Annotation Attribute:** proxyTargetClass
> >
> > **Default:** false
> >
> > **Description:** Applies to proxy mode only. Controls what type of transactional proxies are created for classes annotated with the <font color = red>`@Transactional`</font> annotation. If the <font color = red>`proxy-target-class`</font> attribute is set to <font color = red>`true`</font>, then class-based proxies are created. If <font color = red>`proxy-target-class`</font> is <font color = red>`false`</font> or if the attribute is omitted, then standard JDK interface-based proxies are created. (See [Section 11.6, “Proxying mechanisms”](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop-proxying) for a detailed examination of the different proxy types.)
>
> > **XML Attribute:** order
> >
> > **Annotation Attribute:** order
> >
> > **Default:** Ordered.LOWEST_PRECEDENCE
> >
> > **Description:** Defines the order of the transaction advice that is applied to beans annotated with <font color = red>`@Transactional`</font>. (For more information about the rules related to ordering of AOP advice, see [the section called “Advice ordering”](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop-ataspectj-advice-ordering).) No specified ordering means that the AOP subsystem determines the order of the advice.

> The default advice mode for processing <font color = red>`@Transactional`</font> annotations is "proxy" which allows for interception of calls through the proxy only; local calls within the same class cannot get intercepted that way. For a more advanced mode of interception, consider switching to "aspectj" mode in combination with compile/load-time weaving.

> The <font color = red>`proxy-target-class`</font> attribute controls what type of transactional proxies are created for classes annotated with the <font color = red>`@Transactional`</font> annotation. If<font color = red>`proxy-target-class`</font> is set to <font color = red>`true`</font>, class-based proxies are created. If <font color = red>`proxy-target-class`</font> is <font color = red>`false`</font> or if the attribute is omitted, standard JDK interface-based proxies are created. (See [Section 11.6, “Proxying mechanisms”](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#aop-proxying) for a discussion of the different proxy types.)

> <font color = red>`@EnableTransactionManagement`</font> and <font color = red>`<tx:annotation-driven/>`</font> only looks for <font color = red>`@Transactional`</font> on beans in the same application context they are defined in. This means that, if you put annotation driven configuration in a <font color = red>`WebApplicationContext`</font> for a <font color = red>`DispatcherServlet`</font>, it only checks for <font color = red>`@Transactional`</font> beans in your controllers, and not your services. See [Section 22.2, “The DispatcherServlet”](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#mvc-servlet) for more information.

The most derived location takes precedence when evaluating the transactional settings for a method. In the case of the following example, the <font color = red>`DefaultFooService`</font>class is annotated at the class level with the settings for a read-only transaction, but the <font color = red>`@Transactional`</font> annotation on the <font color = red>`updateFoo(Foo)`</font> method in the same class takes precedence over the transactional settings defined at the class level.

```java
@Transactional(readOnly = true)
public class DefaultFooService implements FooService {

    public Foo getFoo(String fooName) {
        // do something
    }

    // these settings have precedence for this method
    @Transactional(readOnly = false, propagation = Propagation.REQUIRES_NEW)
    public void updateFoo(Foo foo) {
        // do something
    }
}
```

#### @Transactional settings

The  <font color = red>`@Transactional`</font> annotation is metadata that specifies that an interface, class, or method must have transactional semantics; for example, "*start a brand new read-only transaction when this method is invoked, suspending any existing transaction*". The default  <font color = red>`@Transactional`</font> settings are as follows:

- Propagation setting is  <font color = red>`PROPAGATION_REQUIRED.`</font>
- Isolation level is  <font color = red>`ISOLATION_DEFAULT.`</font>
- Transaction is read/write.
- Transaction timeout defaults to the default timeout of the underlying transaction system, or to none if timeouts are not supported.
- Any  <font color = red>`RuntimeException`</font> triggers rollback, and any checked  <font color = red>`Exception`</font> does not.

These default settings can be changed; the various properties of the  <font color = red>`@Transactional`</font> annotation are summarized in the following table:

**Table 17.3. @Transactional Settings**

| Property                                                     | Type                                                         | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [value](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#tx-multiple-tx-mgrs-with-attransactional) | String                                                       | Optional qualifier specifying the transaction manager to be used. |
| [propagation](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#tx-propagation) | enum: `Propagation`                                          | Optional propagation setting.                                |
| `isolation`                                                  | enum: `Isolation`                                            | Optional isolation level. Only applicable to propagation REQUIRED or REQUIRES_NEW. |
| `timeout`                                                    | int (in seconds granularity)                                 | Optional transaction timeout. Only applicable to propagation REQUIRED or REQUIRES_NEW. |
| `readOnly`                                                   | boolean                                                      | Read/write vs. read-only transaction. Only applicable to REQUIRED or REQUIRES_NEW. |
| `rollbackFor`                                                | Array of `Class` objects, which must be derived from `Throwable.` | Optional array of exception classes that *must* cause rollback. |
| `rollbackForClassName`                                       | Array of class names. Classes must be derived from `Throwable.` | Optional array of names of exception classes that *must* cause rollback. |
| `noRollbackFor`                                              | Array of `Class` objects, which must be derived from `Throwable.` | Optional array of exception classes that *must not* cause rollback. |
| `noRollbackForClassName`                                     | Array of `String` class names, which must be derived from `Throwable.` | Optional array of names of exception classes that *must not* cause rollback. |

Currently you cannot have explicit control over the name of a transaction, where 'name' means the transaction name that will be shown in a transaction monitor, if applicable (for example, WebLogic’s transaction monitor), and in logging output. For declarative transactions, the transaction name is always the fully-qualified class name + "." + method name of the transactionally-advised class. For example, if the <font color = red>`handlePayment(..)`</font> method of the <font color = red>`BusinessService`</font> class started a transaction, the name of the transaction would be: <font color = red>`com.foo.BusinessService.handlePayment`</font>.

#### Multiple Transaction Managers with @Transactional

Most Spring applications only need a single transaction manager, but there may be situations where you want multiple independent transaction managers in a single application. The value attribute of the <font color = red>`@Transactional`</font> annotation can be used to optionally specify the identity of the `PlatformTransactionManager` to be used. This can either be the bean name or the qualifier value of the transaction manager bean. For example, using the qualifier notation, the following Java code

```java
public class TransactionalService {

    @Transactional("order")
    public void setSomething(String name) { ... }

    @Transactional("account")
    public void doSomething() { ... }
}
```

could be combined with the following transaction manager bean declarations in the application context.

```xml
<tx:annotation-driven/>

    <bean id="transactionManager1" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="order"/>
    </bean>

    <bean id="transactionManager2" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="account"/>
    </bean>
```

In this case, the two methods on <font color = red>`TransactionalService`</font> will run under separate transaction managers, differentiated by the "order" and "account" qualifiers. The default <font color = red>`<tx:annotation-driven>`</font> target bean name <font color = red>`transactionManager`</font> will still be used if no specifically qualified PlatformTransactionManager bean is found.

#### Custom shortcut annotations

If you find you are repeatedly using the same attributes with `@Transactional` on many different methods, then [Spring’s meta-annotation support](https://docs.spring.io/spring-framework/docs/4.3.21.RELEASE/spring-framework-reference/htmlsingle/#beans-meta-annotations) allows you to define custom shortcut annotations for your specific use cases. For example, defining the following annotations

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional("order")
public @interface OrderTx {
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Transactional("account")
public @interface AccountTx {
}
```

allows us to write the example from the previous section as

```java
public class TransactionalService {

    @OrderTx
    public void setSomething(String name) { ... }

    @AccountTx
    public void doSomething() { ... }
}
```

Here we have used the syntax to define the transaction manager qualifier, but could also have included propagation behavior, rollback rules, timeouts etc.

