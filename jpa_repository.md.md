# 5 JPA 仓库

本章指出JPA建立在上一章内容之上的专有的功能.请确保你对此有充分的了解.

## 5.1 简介

本节介绍通过以下任一方式配置Spring Data JPA的基础知识：
 

- “Spring Namespace” (XML configuration)
- “基于注解的配置” (Java configuration)

### 5.1.1. Spring 命名空间

Spring Data的JPA模块包含一个允许定义存储库bean的自定义命名空间。 它还包含JPA特有的某些功能和元素属性。 通常，可以使用```repositories```元素设置JPA存储库，如以下示例所示：

示例50.使用命名空间设置JPA存储库

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:jpa="http://www.springframework.org/schema/data/jpa"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/data/jpa
    https://www.springframework.org/schema/data/jpa/spring-jpa.xsd">
  <jpa:repositories base-package="com.acme.repositories" />
</beans>
```

使用```repositories```元素查找Spring Data存储库，如“创建存储库实例”中所述。 除此之外，它还激活了使用```@Repository```注释的所有bean的持久性异常转换，以便将JPA持久性提供程序抛出的异常转换为Spring的```DataAccessException```层次结构。

#### 用户命令空间属性 

除了存储库元素的默认属性之外，JPA命名空间还提供了其他属性，使您可以更好地控制存储库的设置：

表2.存储库元素的自定义JPA特定属性

|属性|描述|
|---|---|
|entity-manager-factory-ref|显式连接要与存储库元素检测到的存储库一起使用的```EntityManagerFactory```。 通常在应用程序中使用多个```EntityManagerFactory``` bean时使用。 如果未配置，Spring Data会自动在```ApplicationContext```中使用名称```entityManagerFactory```查找```EntityManagerFactory``` bean。|
|transaction-manager-ref|显式连接```PlatformTransactionManager```以与存储库元素检测到的存储库一起使用。 通常仅在配置了多个事务管理器或```EntityManagerFactory``` bean时才需要。 默认为当前```ApplicationContext```中单个定义的```PlatformTransactionManager```。|

>如果没有定义显式的```transaction-manager-ref```，Spring Data JPA需要一个名为```transactionManager```的```PlatformTransactionManager``` bean。


### 5.1.2. 基于注解的配置

Spring Data JPA存储库支持不仅可以通过XML命名空间激活，还可以通过JavaConfig使用注释来激活，如以下示例所示：

例51.使用JavaConfig的Spring Data JPA存储库

```java
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
class ApplicationConfig {

  @Bean
  public DataSource dataSource() {

    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    return builder.setType(EmbeddedDatabaseType.HSQL).build();
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {

    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setGenerateDdl(true);

    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setJpaVendorAdapter(vendorAdapter);
    factory.setPackagesToScan("com.acme.domain");
    factory.setDataSource(dataSource());
    return factory;
  }

  @Bean
  public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {

    JpaTransactionManager txManager = new JpaTransactionManager();
    txManager.setEntityManagerFactory(entityManagerFactory);
    return txManager;
  }
}
```

>您必须直接创建```LocalContainerEntityManagerFactoryBean```而不是```EntityManagerFactory```，因为除了创建```EntityManagerFactory```之外，前者还参与异常转换机制。

上述配置类使用spring-jdbc的```EmbeddedDatabaseBuilder``` API设置嵌入式HSQL数据库。 然后Spring Data设置一个```EntityManagerFactory```并使用Hibernate作为示例持久性提供程序。 这里声明的最后一个基础架构组件是```JpaTransactionManager```。 最后，该示例使用```@EnableJpaRepositories```注释激活Spring Data JPA存储库，该注释基本上具有与XML命名空间相同的属性。 如果未配置基本软件包，则使用配置类所在的基础软件包。

###  5.1.3. 启动模式

默认情况下，Spring Data JPA存储库是默认的Spring bean。他们是单例并会尽快初始化.在启动期间.他们已经与JPA 的 ```EntityManager```进行交互以进行验证和元数据分析.Spring框架支持在后台线程中初始化JPA ```EntityManager```,因为该进程通常在Spring应用程序中占用大量的启动时间.为了有效的利用后台初始化.我们需要尽可能晚的初始化JPA数据库.

从Spring Data JPA 2.1 开始你可以通过 ```@EnableJpaRepositories```注解和 ```XML```配置```BootstrapMode``` 为如下值:

- ```DEFAULT```(默认值) - 除非用```@Lazy``` 显式注释，否则存储库将被尽快的实例化。只有当没有客户机bean需要存储库实例时，延迟才有效，因为这将需要存储库bean的初始化。 
- ```LAZY``` — 隐式声明所有存储库bean为懒加载，还导致创建的lazy初始化代理被注入到客户端bean中。这意味着，如果客户机bean只是将实例存储在一个字段中，并且在初始化期间没有使用存储库，那么存储库将不会被实例化。存储库实例将在第一次与存储库交互时初始化和验证。
- ```DEFERRED``` — 基本上与lazy的操作模式相同，但会触发存储库初始化以响应```contextRefreshedEvent```，以便在应用程序完全启动之前验证存储库。


##### 建议

如果您没有使用默认引导程序模式的异步JPA引导程序棒。

如果您异步引导JPA，```DEFERRED```是一个合理的默认值，因为它将确保Spring Data JPA引导程序仅等待```EntityManagerFactory```设置，如果它本身比初始化所有其他应用程序组件花费更长时间。 但是，它确保在应用程序发出信号之前正确初始化和验证存储库。

```LAZY```是测试场景和本地开发的不错选择。 一旦您非常确定存储库将正确引导，或者在您测试应用程序的其他部分的情况下，对所有存储库执行验证可能只会不必要地增加启动时间。 这同样适用于本地开发，在该开发中，您只访问可能只需要初始化单个存储库的应用程序部分。

## 5.2 持久化实体

本节描述如何使用spring data jpa持久化（保存）实体 

### 5.2.1 保存实体

可以使用```CrudRepository.save（…）``` 方法保存实体。它通过使用底层JPA  ```EntityManager```来持久化或合并给定的实体。如果实体还没有被持久化，Spring Data JPA会通过调用```entityManager.persist（…）```方法来保存实体。否则，它将调用```entityManager.merge（…）```方法。


#### 实体状态检测策略

Spring Data JPA 提供了以下策略来检测实体是否是新的：

- ID属性检查（默认）：默认情况下，Spring Data JPA 检查给定实体的identifier属性。如果identifier属性为空，则假定实体是新的。否则，它被认为不是新的。 
- 实现持久化：如果一个实体实现持久化，Spring Data JPA 将新的检测委托给实体的```isNew（…）```方法。有关详细信息，请参见[javaDoc](https://docs.spring.io/spring-data/data-commons/docs/current/api/index.html?org/springframework/data/domain/Persistable.html)。

- 实现```EntityInformation```：通过创建jparepositoryfactory的子类并相应地重写```getEntityInformation（…）```方法，可以自定义```SimpleJpaRepository```实现中使用的```EntityInformation```抽象。然后必须将```JpaRepositoryFactory```的自定义实现注册为Spring Bean。注意，这应该是很少必要的。有关详细信息，请参见[javadoc](https://docs.spring.io/spring-data/data-jpa/docs/current/api/index.html?org/springframework/data/jpa/repository/support/JpaRepositoryFactory.html)。


## 5.3 查询方法

本节介绍使用Spring Data JPA 创建查询的各种方法。

### 5.3.1 查询查找策略 

JPA模块支持将查询手动定义为字符串或从方法名派生查询。 

带有谓词的派生查询```IsStartingWith```，```StartingWith```，```StartsWith```，```IsEndingWith```，```EndingWith```，```EndsWith```，```IsNotContaining```，```NotContaining```，```NotContains```，```IsContaining```，```Containing```，包含这些查询的相应参数将被清理。这意味着如果参数包含```LIKE```识别的字符作为通配符，这些将被转义，它们只匹配为文字。可以通过设置```@EnableJpaRepositories```注释的```escapeCharacter```来配置使用的转义字符。[与使用SpEL表达式进行比较](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query.spel-expressions)。


#### 声明查询

尽管获取从方法名称派生的查询非常方便，但是可能面临这样的情况：方法名称解析器不支持要使用的关键字，或者方法名称会变得不那么优雅。 因此您可以通过命名约定使用JPA命名查询（[请参阅使用JPA命名查询获取更多信息](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.named-queries)），或者使用```@Query```注释查询方法（有关详细信息，请参阅使用@Query）。


### 5.3.2. 查询的创建

通常,JPA查询的创建机制与第四章的描述差不多. 以下示例显示了JPA查询方法转换为的内容：

示例52.从方法名称创建查询

```java
public interface UserRepository extends Repository<User, Long> {
  List<User> findByEmailAddressAndLastname(String emailAddress, String lastname);
}
```








