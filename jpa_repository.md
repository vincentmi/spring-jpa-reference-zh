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

> 我们使用JPA标准API创建一个查询，但实质上，这个转换为以下查询：```select u from User u where u.emailAddress = ?1 and u.lastname = ?2```。 Spring Data JPA执行属性检查并遍历嵌套属性，请参考上面的的属性表达式。

下表描述了JPA支持的关键字以及包含该关键字的方法转换为：


| 关键字 |	示例 	| JPQL 片段|
|---|---|---|
|And|findByLastnameAndFirstname|… where x.lastname = ?1 and x.firstname = ?2|
|Or|findByLastnameOrFirstname|… where x.lastname = ?1 or x.firstname = ?2|
|Is,Equals|findByFirstname,findByFirstnameIs,findByFirstnameEquals|… where x.firstname = ?1|
|Between|findByStartDateBetween|… where x.startDate between ?1 and ?2|
|LessThan|findByAgeLessThan|… where x.age < ?1|
|LessThanEqual|findByAgeLessThanEqual|… where x.age <= ?1|
|GreaterThan|findByAgeGreaterThan|… where x.age > ?1|
|GreaterThanEqual|findByAgeGreaterThanEqual|… where x.age >= ?1|
|After|findByStartDateAfter|… where x.startDate > ?1|
|Before|findByStartDateBefore|… where x.startDate < ?1|
|IsNull|findByAgeIsNull|… where x.age is null|
|IsNotNull,NotNull|findByAge(Is)NotNull|… where x.age not null|
|Like|findByFirstnameLike|… where x.firstname like ?1|
|NotLike|findByFirstnameNotLike|… where x.firstname not like ?1|
|StartingWith|findByFirstnameStartingWith|… where x.firstname like ?1 (parameter bound with appended %)|
|EndingWith|findByFirstnameEndingWith|… where x.firstname like ?1 (parameter bound with prepended %)|
|Containing|findByFirstnameContaining|… where x.firstname like ?1 (parameter bound wrapped in %)|
|OrderBy|findByAgeOrderByLastnameDesc|… where x.age = ?1 order by x.lastname desc|
|Not|findByLastnameNot|… where x.lastname <> ?1|
|In|findByAgeIn(Collection<Age> ages)|… where x.age in ?1|
|NotIn|findByAgeNotIn(Collection<Age> ages)|… where x.age not in ?1|
|True|findByActiveTrue()|… where x.active = true|
|False|findByActiveFalse()|… where x.active = false|
|IgnoreCase|findByFirstnameIgnoreCase|… where UPPER(x.firstame) = UPPER(?1)|


> ```In```和```NotIn```可以使用```Collection```的任何子类作为参数以及参数数组。 对于同一逻辑运算符的其他语法版本，请选中“存储库查询关键字”。


### 5.3.3 使用JPA命名查询

> 这些示例使用```<named-query />```元素和```@NamedQuery```注解。 必须在JPA查询语言中定义这些配置元素的查询。 当然，您也可以使用```<named-native-query />```或```@NamedNativeQuery```。 这些元素允许您通过失去数据库平台独立性来在使用原生SQL中定义查询。

#### XML命名查询定义

要使用XML配置，请将必要的```<named-query />```元素添加到位于类路径的```META-INF```文件夹中的```orm.xml``` JPA配置文件中。 通过使用某些已定义的命名约定，可以自动调用命名查询。 有关详细信息，请参阅下文。

例53. XML命名查询配置

```xml
<named-query name="User.findByLastname">
  <query>select u from User u where u.lastname = ?1</query>
</named-query>
```

这个查询有一个特定的名称,用于在运行时进行查找.

### 基于注解的配置

基于注解的配置具有不需要编辑另一个配置文件的优点，从而降低了维护工作量。 您需要为每个新的查询声明重新编译域类，从而为此获益。

示例54.基于注释的命名查询配置

```java
@Entity
@NamedQuery(name = "User.findByEmailAddress",
  query = "select u from User u where u.emailAddress = ?1")
public class User {

}
```

#### 定义接口

要允许执行这些命名查询，请按如下所示指定```UserRepository```：

例55. ```UserRepository```中的查询方法声明

```java
public interface UserRepository extends JpaRepository<User, Long> {

  List<User> findByLastname(String lastname);

  User findByEmailAddress(String emailAddress);
}
```

Spring Data尝试将对这些方法的调用解析为命名查询，从配置的域类的简单名称开始，后跟由点分隔的方法名称。 因此，前面的示例将使用在examlpe中定义的命名查询，而不是尝试从方法名称创建查询。

### 5.3.4 使用```@Query```

使用命名查询来声明实体查询是一种有效的方法，适用于少量查询。 由于查询本身与执行它们的Java方法相关联，因此您实际上可以使用Spring Data JPA ```@Query```注释直接绑定它们，而不是将它们注释到域类。 这将域类从特定于持久性的信息中释放出来，并将查询与存储库接口共同定位。

对查询方法进行注解的查询优先于使用```@namedQuery```定义的查询或在```orm.xml```中声明的命名查询。

以下示例显示使用```@Query```注释创建的查询：

例56.使用```@Query```在查询方法中声明查询

```java
public interface UserRepository extends JpaRepository<User, Long> {
  @Query("select u from User u where u.emailAddress = ?1")
  User findByEmailAddress(String emailAddress);
}
```

#### 使用高级的 ```LIKE``` 表达式

使用```@Query```创建的手动定义查询的查询执行机制允许在查询定义中定义高级```LIKE```表达式，如以下示例所示：

例57. ```@Query```中的高级类似表达式

```java
public interface UserRepository extends JpaRepository<User, Long> {
  @Query("select u from User u where u.firstname like %?1")
  List<User> findByFirstnameEndsWith(String firstname);
}
```

在前面的示例中，识别了```LIKE```的分隔符```%```，并将查询转换为有效的```JPQL```查询（删除```％```）。 在执行查询时，传递给方法调用的参数将使用先前识别的```LIKE```模式进行扩充。


####  源生查询

```@Query```注释允许通过将```nativeQuery```标志设置为```true```来运行源生查询，如以下示例所示：

例58.使用@Query在查询方法中声明本机查询

```java
public interface UserRepository extends JpaRepository<User, Long> {
  @Query(value = "SELECT * FROM USERS WHERE EMAIL_ADDRESS = ?1", nativeQuery = true)
  User findByEmailAddress(String emailAddress);
}
```

Spring Data JPA目前不支持对源生查询进行动态排序，因为它必须操纵声明的实际查询，而对于源生SQL，它无法可靠地执行。 但是，您可以通过自己指定计数查询来使用本机查询进行分页，如以下示例所示：

例59.使用@Query在查询方法中声明分页的本机计数查询

```java
public interface UserRepository extends JpaRepository<User, Long> {
  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

类似的方法也适用于命名的源生查询，方法是将```.count```后缀添加到查询副本中。 但是，您可能需要为计数查询注册结果集映射。

### 5.3.5 使用排序

可以通过提供```PageRequest```或直接使用```Sort```来完成排序。 ```Sort```实例的```Order```属性的取值需要与您的领域模型的属性相匹配，这意味着它们需要解析为查询中使用的属性或别名。 JPQL将其定义为状态字段路径表达式。

> 使用任何不可引用的路径表达式会导致异常。

但是，使用```Sort```与```@Query```一起使用可以让您潜入包含```ORDER BY```子句中的函数的非路径检查的```Order```实例。 因为Order被附加到给定的查询字符串。 默认情况下，Spring Data JPA拒绝任何包含函数调用的```Order```实例，但您可以使用```JpaSort.unsafe```添加可能不安全的顺序。

以下示例使用```Sort```和```JpaSort```，包括```JpaSort```上的不安全选项：

例60.使用Sort和JpaSort
```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.lastname like ?1%")
  List<User> findByAndSort(String lastname, Sort sort);

  @Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
  List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);
}

repo.findByAndSort("lannister", new Sort("firstname"));       //1         
repo.findByAndSort("stark", new Sort("LENGTH(firstname)"));   //2     
repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)")); //3 
repo.findByAsArrayAndSort("bolton", new Sort("fn_len"));              //4
```

1. 有效的```Sort```,表达式指向一个领域模型的属性.
2. 无效的 ```Sort``` 包含了函数,抛出异常
3. 有效的 ```Sort``` 包含了 ```unsafe```的 ```Order```
4. 有效的 ```Sort``` 包含了指向别名的函数

### 5.3.6 使用命名的参数

默认情况下，Spring Data JPA使用基于位置的参数绑定，如前面所有示例中所述。 这使得查询方法在重构参数位置时容易出错。 要解决此问题，可以使用```@Param```注解为方法参数指定具体名称并在查询中绑定名称，如以下示例所示：

例61.使用命名参数

```java
public interface UserRepository extends JpaRepository<User, Long> {
  @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
  User findByLastnameOrFirstname(@Param("lastname") String lastname,
                                 @Param("firstname") String firstname);
}
```

> 方法参数根据它们在定义的查询中的顺序进行切换。
> 
> 从版本4开始，Spring完全支持基于-parameters编译器标志的Java 8参数名称发现。 通过在构建中使用此标志作为调试信息的替代方法，可以省略命名参数的@Param注释。
> 

### 5.3.7 使用SpEL表达式

从Spring Data JPA 1.4版开始，我们支持在使用```@Query```定义的手动定义查询中使用受限制的```SpEL```模板表达式。 在执行查询时，将根据预定义的变量集评估这些表达式。 Spring Data JPA支持名为```entityName```的变量。 它的用法是```select x from #{#entityName} x```。 它插入与给定存储库关联的域类型的```entityName```。 ```entityName```的解析方式如下：如果域类型在```@Entity```注释上设置了```name```属性，则使用它。 否则，使用域类型的简单类名。


以下示例演示了查询字符串中```＃{#entityName}```表达式的一个用例，您希望使用查询方法和手动定义的查询来定义存储库接口：

示例62.在存储库查询方法中使用SpEL表达式 -  entityName

```java
@Entity
public class User {
  @Id
  @GeneratedValue
  Long id;
  String lastname;
}

public interface UserRepository extends JpaRepository<User,Long> {
  @Query("select u from #{#entityName} u where u.lastname = ?1")
  List<User> findByLastname(String lastname);
}
```

要避免在```@Query```注解的查询字符串中声明实际的实体名称，可以使用```＃{#entityName}```变量。

> 可以使用```@Entity```批注自定义```entityName```。 SpEL表达式不支持```orm.xml``中自定义的名称。

当然，您可以直接在查询声明中使用```User```，但这需要你来更高查询声明。 对```#entityName```的引用将```User```类的可以将他在需要的时候映射到另外一个实体（例如，使用```@Entity（name =“MyUser”）```）。

查询字符串中```＃{#entityName}```表达式的另一个用例是，如果要为具体领域类型定义具有专用存储库接口的通用存储库接口。 要在具体接口上不重复自定义查询方法的定义，可以在通用存储库接口中的```@Query```批注的查询字符串中使用实体名称表达式，如以下示例所示：

示例63.在存储库查询方法中使用SpEL表达式 - 具有继承的entityName

```java
@MappedSuperclass
public abstract class AbstractMappedType {
  …
  String attribute
}
@Entity
public class ConcreteType extends AbstractMappedType { … }
@NoRepositoryBean
public interface MappedTypeRepository<T extends AbstractMappedType>
  extends Repository<T, Long> {
  @Query("select t from #{#entityName} t where t.attribute = ?1")
  List<T> findAllByAttribute(String attribute);
}

public interface ConcreteRepository
  extends MappedTypeRepository<ConcreteType> { … }
```

In the preceding example, the MappedTypeRepository interface is the common parent interface for a few domain types extending AbstractMappedType. It also defines the generic findAllByAttribute(…) method, which can be used on instances of the specialized repository interfaces. If you now invoke findByAllAttribute(…) on ConcreteRepository, the query becomes select t from ConcreteType t where t.attribute = ?1.

在前面的示例中，```MappedTypeRepository```接口是扩展```AbstractMappedType```的几种领域类型的公共父接口。 它还定义了通用的```findAllByAttribute（...）```方法，该方法可用于专用存储库接口的实例。 如果现在在```ConcreteRepository```上调用```findByAllAttribute（...）```，则查询变为 ```select t from ConcreteType t where t.attribute = ?1```.

操作参数的SpEL表达式也可用于操作方法参数。 在这些SpEL表达式中，实体名称不能适应，但参数可以使用。 可以通过名称或索引访问它们，如以下示例所示。

示例64.在存储库查询方法中使用SpEL表达式 - 访问参数。

```java
@Query("select u from User u where u.firstname = ?1 and u.firstname=?#{[0]} and u.emailAddress = ?#{principal.emailAddress}")
List<User> findByFirstnameAndCurrentUserWithCustomQuery(String firstname);
```

对于```LIKE```条件，人们通常希望将```％```附加到字符串值参数的开头或结尾。 这可以通过使用```％```附加到参数标记上或SpEL表达式来完成。 以下示例再次演示了这一点。

示例65.在存储库查询方法中使用Spel表达式 - 通配符快捷方式。

```java
@Query("select u from User u where u.lastname like %:#{[0]}% and u.lastname like %:lastname%")
List<User> findByLastnameWithSpelExpression(@Param("lastname") String lastname);
```

当使用Like条件时,如果数据源是不安全的,我们应该对值进行清理避免他们使用通配符,让攻击者获取到超出许可范围的数据.为此SpEL上下文中提供了```escape(string)```方法.它使用第二个参数中的单个字符为第一个参数中```_```和```％```的所有实例添加前缀,结合JPQL和标准SQL中提供的类似表达式的escape子句，可以轻松清理绑定的参数。 

示例66.在存储库查询方法中使用SpEL表达式 - 清理输入值。

```java
@Query("select u from User u where u.firstname like %?#{escape([0])}% escape ?#{escapeCharacter()}")
List<User> findContainingEscaped(String namePart);
```

上例的接口方法```findContainingEscaped("Peter_")```将会找到```Peter_Parker```但是不会找到```Peter Parker```.可以通过设置```@EnableJpaRepositories ```注解的```escapeCharacter```来配置要使用的转义符.请注意,方法```escape(string)```可以用在```SpEL```上下文中只会转义SQL和JPQL标准通配符```_```和```%```,如果底层数据库或JPA的实现支持其他通配符.则这些通配符不会被转义.

### 5.3.8 修改语句

前面的所有部分都描述了如何声明查询以访问给定的实体或实体集合。您可以使用“Spring数据存储库的自定义实现”中描述的工具添加自定义修改行为。由于这种方法可以很自由的进行自定义，您可以通过使用```@Modify```注释查询方法来修改只需要参数绑定的查询，如下例所示：

例67 声明操作查询

```java
@Modifying
@Query("update User u set u.firstname = ?1 where u.lastname = ?2")
int setFixedFirstnameFor(String firstname, String lastname);
```
这样做会将查询定义为更新而不是查询.由于```EntityManager```在更新执行后会包含过期的实体.我们并不会自动清理他们.因为这实际上会删除```EntityManager```中等待更新的更改.如果希望自动清除```EntityManager```中的这些实体.可以将```@Modifying ```注解的```clearAutomatically```属性设置为```true```.

```@Modifying```注释只与```@Query```注释结合使用。派生查询方法或自定义方法不需要此批注。

#### 派生查询语句

Spring Data JPA 还支持派生的删除语句，这样可以避免显式声明JPSQL查询，如下例所示：

例68。使用派生的删除查询 

```java
interface UserRepository extends Repository<User, Long> {
  void deleteByRoleId(long roleId);
  @Modifying
  @Query("delete from User u where user.role.id = ?1")
  void deleteInBulkByRoleId(long roleId);
}
```


尽管```deleteByroleid（…）```方法看起来基本上生成的结果与```deleteBulkByRoleID（…）```相同，但这两个方法声明在执行方式上有一个重要区别。顾名思义，后一种方法针对数据库发出一个JPQL查询（在注释中定义的查询）。这意味着即使当前加载的User实例也看不到调用的生命周期回调。 


为了确保生命周期查询被实际调用，对```deleteByroleid（…）```的调用执行一个查询，然后逐个删除返回的实例，以便持久性提供程序可以实际调用这些实体上的删除前的回调。


事实上，派生的delete查询是执行查询，然后对结果调用```CrudRepository.delete(Iterable<User> users) ```并使行为与```CrudRepository```中其他```delete（…）```方法的实现保持同步的快捷方式。

### 5.3.9. 应用查询提示

要将JPA查询提示应用于存储库接口中声明的查询，可以使用```@QueryHints```批注。 它需要一组JPA ```@QueryHint```注解以及一个布尔标志来潜在地禁用应用于应用分页时触发的附加计数查询的提示，如以下示例所示：

示例69.将QueryHints与存储库方法一起使用

```java
public interface UserRepository extends Repository<User, Long> {

  @QueryHints(value = { @QueryHint(name = "name", value = "value")},forCounting = false)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

前面的声明将为实际查询应用配置的```@QueryHint```，但并不应用到计数查询中。 

### 5.3.10 配置 Fetch- 和 LoadGraphs

JPA 2.1规范引入了对指定Fetch-和LoadGraphs的支持，我们也支持```@EntityGraph```注解，它允许您引用```@NamedEntityGraph```定义。 您可以在实体上使用该注解来配置生成的查询的获取计划。 可以使用```@EntityGraph```注解上的```type```属性配置提取的类型（Fetch或load）。 有关进一步的参考，请参阅JPA 2.1 Spec 3.7.4。

以下示例显示如何在实体上定义命名实体图：

示例70.在实体上定义命名实体图。

```java
@Entity
@NamedEntityGraph(name = "GroupInfo.detail",
  attributeNodes = @NamedAttributeNode("members"))
public class GroupInfo {
  // default fetch mode is lazy.
  @ManyToMany
  List<GroupMember> members = new ArrayList<GroupMember>();
  …
}
```

以下示例显示如何在存储库查询方法上引用命名实体图：

例71.在存储库查询方法上引用命名实体图定义。

```java 
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {
  @EntityGraph(value = "GroupInfo.detail", type = EntityGraphType.LOAD)
  GroupInfo getByGroupName(String name);
}
```

也可以使用```@EntityGraph```定义ad hoc实体图。 提供的```attributePaths```将转换为相应的```EntityGraph```，而无需将```@NamedEntityGraph```显式添加到您的域类型中，如以下示例所示：

例子72.在存储库查询方法上使用AD-HOC实体图定义。

```java
@Repository
public interface GroupRepository extends CrudRepository<GroupInfo, String> {
  @EntityGraph(attributePaths = { "members" })
  GroupInfo getByGroupName(String name);
}
```

### 5.3.11.投影

Spring Data查询方法通常返回由存储库管理的聚合根的一个或多个实例。 但是，有时可能需要根据这些类型的某些属性创建投影。 Spring Data允许建模专用返回类型，以更有选择地检索托管聚合的部分视图。

假设有  一个存储库和聚合根类型，如以下示例：

```java
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {
  Collection<Person> findByLastname(String lastname);
}
```

假设我们只想检索这个人的名字属性。Spring Data 为实现这一目标提供了什么手段？这一章的其余部分回答了这个问题。

#### 基于接口的投影

将查询结果限制为只返回名称属性的最容易的方式是如下例所示,定义一个只暴露你需要的属性的getter方法的接口:

例74。检索属性子集的投影接口

```java
interface NamesOnly {
  String getFirstname();
  String getLastname();
}
```

比较重要的一点是,这里定义的属性与聚合根的属性需要完全匹配.然后使用下面的方式定义查询:

例75。使用基于接口的投影和查询方法的存储库

```java
interface PersonRepository extends Repository<Person, UUID> {
  Collection<NamesOnly> findByLastname(String lastname);
}
```

查询执行引擎在运行时为返回的每个元素创建该接口的代理实例，并将调用转发到目标对象的公开方法。

投影可以递归使用。如果还希望包含一些地址信息，请为此创建一个投影接口，并从```getaddress（）```的声明中返回该接口，如下例所示：

例76。检索属性子集的投影接口

```java
interface PersonSummary {
  String getFirstname();
  String getLastname();
  AddressSummary getAddress();
  interface AddressSummary {
    String getCity();
  }
}
```

在方法调用时，获取目标实例的address属性，并依次将其包装到投影代理中。

#### 闭合投影
一个投影接口，其访问器方法都与目标聚合的属性匹配，被认为是一个闭合投影。以下示例（我们在本章前面也使用了该示例）是一个闭合投影：

```java
interface NamesOnly {
  String getFirstname();
  String getLastname();
}
```

如果使用封闭投影，Spring Data 可以优化查询的执行，因为我们知道支持投影代理所需的所有属性。有关更多详细信息，请参阅参考文档中特定于模块的部分。

#### 开放投影

投影接口中的访问器方法也可以使用```@Value```注释计算新值，如下例所示

```java
interface NamesOnly {

  @Value("#{target.firstname + ' ' + target.lastname}")
  String getFullName();
  …
}
```

```target```变量中可以使用聚合根的所以属性。使用```@Value```的投影接口是一个开放的投影。在这种情况下，Spring Data 不能应用查询执行优化，因为SpEL表达式可以使用聚合根的所有属性。

```@Value```中使用的表达式不应太复杂-要避免在字符串变量中编程。对于非常简单的表达式，也可以使用JAVA 8 中引入的默认方法，如下面的示例所示：

```java
interface NamesOnly {

  String getFirstname();
  String getLastname();
  default String getFullName() {
    return getFirstname.concat(" ").concat(getLastname());
  }
}
```

这种方法要求您能够完全基于投影接口上公开的其他访问器方法来实现逻辑。第二种方法是在Spring Bean中实现自定义逻辑，然后从SpEL表达式调用该逻辑，如下例所示：

```java
@Component
class MyBean {

  String getFullName(Person person) {
    …
  }
}

interface NamesOnly {
  @Value("#{@myBean.getFullName(target)}")
  String getFullName();
  …
}
```

注意SpEL表达式如何引用```myBean```并调用```getfullname（…）```方法并将投影目标作为方法参数转发。由SpEL表达式求值支持的方法也可以使用方法参数，然后可以从表达式引用这些参数。方法参数可通过名为```args```的对象数组获得。下面的示例演示如何从```args```数组中获取方法参数

```java
interface NamesOnly {
  @Value("#{args[0] + ' ' + target.firstname + '!'}")
  String getSalutation(String prefix);
}
```

同样,如前所述，对于更复杂的表达式，应该使用Spring Bean 并让表达式调用Bean的方法.


#### 基于Class的投影(DTOs)

定义投影的另一种方法是使用值类型DTO（Data Transfer Objects），该DTO保存应该检索的字段的属性。这些DTO类型的使用方式与使用投影接口的方式完全相同，只是不会发生代理，也不会应用嵌套投影。


如果store通过限制要加载的字段来优化查询执行，则要加载的字段由公开的构造函数的参数名确定。

```java
class NamesOnly {

  private final String firstname, lastname;

  NamesOnly(String firstname, String lastname) {
    this.firstname = firstname;
    this.lastname = lastname;
  }

  String getFirstname() {
    return this.firstname;
  }

  String getLastname() {
    return this.lastname;
  }

  // equals(…) and hashCode() implementations
}
```

##### 避免投影DTO的样板代码
 通过使用project lombok，您可以极大地简化DTO的代码，projectlombok提供了一个```@Value```注释（不要与前面接口示例中显示的spring的```@Value```注释混淆）。如果使用project lombok的```@Value```注释，前面显示的示例DTO将变成以下内容：

```java
 @Value
class NamesOnly {
	String firstname, lastname;
}
```

默认情况下，字段是```private final```，class 公开一个构造函数，该构造函数接受所有字段，并自动获取实现的```equals（…）```和```hashcode（）```方法。


#### 动态投影

到目前为止，我们使用投影类型作为集合的返回类型或元素类型。但是，您可能希望选择在调用时使用的类型（这使其成为动态的）。要应用动态投影，请使用以下示例中所示的查询方法：

```java
interface PersonRepository extends Repository<Person, UUID> {
  <T> Collection<T> findByLastname(String lastname, Class<T> type);
}
```

这样，该方法可用于按原样或使用投影获得骨料，如下例所示：

```java
void someMethod(PersonRepository people) {

  Collection<Person> aggregates =
    people.findByLastname("Matthews", Person.class);

  Collection<NamesOnly> aggregates =
    people.findByLastname("Matthews", NamesOnly.class);
}
```

## 5.4 存储过程

JPA 2.1规范引入了对使用JPA条件查询API调用存储过程的支持。我们引入了```@Procedure```注释，用于仓库方法上声明存储过程元数据。



例85。HSQL DB中```plus1inout```过程的定义。

```sql
/;
DROP procedure IF EXISTS plus1inout
/;
CREATE procedure plus1inout (IN arg int, OUT res int)
BEGIN ATOMIC
 set res = arg + 1;
END
/;
```

存储过程的元数据可以使用```@NamedStoredProcedureQuery```对实体类型注解进行设置.

```java
@Entity
@NamedStoredProcedureQuery(name = "User.plus1", procedureName = "plus1inout", parameters = {
  @StoredProcedureParameter(mode = ParameterMode.IN, name = "arg", type = Integer.class),
  @StoredProcedureParameter(mode = ParameterMode.OUT, name = "res", type = Integer.class) })
public class User {}
```

您可以通过多种方式引用存储库方法中的存储过程。要调用的存储过程可以使用```@Procedure```注释的```value```或```procedureName```属性直接定义，也可以使用```name```属性间接定义。如果未配置名称，则使用存储库方法的名称。

下面的示例演示如何引用显式映射的过程：

```java
@Procedure("plus1inout")
Integer explicitlyNamedPlus1inout(Integer arg);
```

下面的示例演示如何使用```procedureName```别名引用隐式映射的过程：

```java
@Procedure(procedureName = "plus1inout")
Integer plus1inout(Integer arg);
```


下面的示例演示如何在```EntityManager```中引用显式映射的命名过程：

```java
@Procedure(name = "User.plus1IO")
Integer entityAnnotatedCustomNamedProcedurePlus1IO(@Param("arg") Integer arg);
```

下面的示例演示如何使用方法名引用EntityManager中隐式命名的存储过程：

```java
@Procedure
Integer plus1(@Param("arg") Integer arg);
```

## 5.5 Specification

JPA 2 引入了一个标准API,你可以使用它以编程的方式构建查询.通过编写```criteria```你定义领域类的查询的```where```子句.再退一步，可以将这些条件视为JPA条件API约束所描述的实体上的谓词。

Spring JPA 采用了 Eric Evans 的 <领域驱动设计>(DDD) 书中的概念,遵循相同的语义,并提供JPA crieria API.为了支持这个规范你可以将你的仓库接口扩展自```JpaSpecificationExecutor```,如下例所示:

```java
public interface CustomerRepository extends CrudRepository<Customer, Long>, JpaSpecificationExecutor {
 …
}
```

附加的接口有一些方法，可以让您以各种方式执行规范。例如，```findAll```方法返回与规范匹配的所有实体，如下例所示：

```java
List<T> findAll(Specification<T> spec);
```

```Specification``` 接口定义如下:

```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder);
}
```

规范可以很容易的在实体之上构建一组可扩展的谓词,然后可以与```JpaRepositiry```组合使用,而无需为每个所需的组合声明方法.如下所示

```java
public class CustomerSpecs {

  public static Specification<Customer> isLongTermCustomer() {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         LocalDate date = new LocalDate().minusYears(2);
         return builder.lessThan(root.get(_Customer.createdAt), date);
      }
    };
  }

  public static Specification<Customer> hasSalesOfMoreThan(MonetaryAmount value) {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         // build query here
      }
    };
  }
}
```

诚然，样板的代码还有改进的空间（可以使用Java 8的闭包）减少，但是客户代码会变得更好，如你在本节后面会看到的。```_Customer```类型是使用JPA元模型生成器生成的元模型类型（请参阅hibernate实现的文档中的示例）。因此，表达式```_Customer.createdAt```假定```Customer```具有```Date```类型的```createdAt```属性。除此之外，我们在业务需求抽象层上表达了一些标准，并创建了可执行规范。因此，客户可以使用以下规范：

```java
List<Customer> customers = customerRepository.findAll(isLongTermCustomer());
```


为什么不为这种数据访问创建一个查询呢？使用单个```Specification```并不能比简单的查询声明获得很多好处。当您将```Specification```组合起来创建新的```Specification```对象时，```Specification```的威力就非常强大。您可以通过我们提供的默认```Specification```方法来实现这一点，这些方法用于构建类似于以下内容的表达式：

```java
MonetaryAmount amount = new MonetaryAmount(200.0, Currencies.DOLLAR);
List<Customer> customers = customerRepository.findAll(
  isLongTermCustomer().or(hasSalesOfMoreThan(amount)));
```

```Specification```提供一些胶合代码的默认方法来链接和组合```Specification```实例.这些方法允许您通过创建新的```Specification```实现并将它们与现有实现相结合来扩展数据访问层。


## 使用 Example 进行查询

### 5.6.1 简介

这一节介绍使用```Example```进行查询并解释如何使用它.

示例查询（QBE）是一种用户友好的查询技术，具有简单的界面。它允许动态创建查询，并且不需要您编写包含字段名的查询。事实上，按QBE根本不需要使用特定于存储的查询语言编写查询。

### 5.6.2 使用

```Example```查询分为三部分

- Probe : 带有填充字段的域对象的实际实例
- ExampleMatcher : ExampleMatcher包含如何匹配特点字段的详细信息,他可以跨多个示例重用.
- Example: 有```Probe```和```ExampleMatcher```组成.用于生成查询.

```Example``` 查询适用一下场景

- 使用一组静态或动态约束查询数据存储。
- 经常重构域对象，而不必担心破坏现有查询。
- 独立于底层数据存储api工作。

```Example``` 查询的一些限制
- 不支持嵌套或分组的属性约束，例如  firstname = ?0 或者 (firstname = ?1 and lastname = ?2).
- 仅支持字符串的```start```/```contains```/```ends```/```regex```匹配和其他属性类型的精确匹配。

在开始按示例查询之前，您需要有一个域对象。要开始，请为存储库创建一个接口，如下例所示：
 
```java
public class Person {

  @Id
  private String id;
  private String firstname;
  private String lastname;
  private Address address;

  // … getters and setters omitted
}
```


The preceding example shows a simple domain object. You can use it to create an Example. By default, fields having null values are ignored, and strings are matched by using the store specific defaults. Examples can be built by either using the of factory method or by using ExampleMatcher. Example is immutable. The following listing shows a simple Example:
前面的示例展示了一个简单的域对象。你可以用它来创建一个```Example```。默认情况下，将忽略具有空值的字段，并使用特定于存储区的默认值来匹配字符串。```Example```可以使用工厂方法或```ExampleMatcher```来构建示例。```Example```是不可变的。下面的列表显示了一个简单的示例：

```java
Person person = new Person();  //(1)                        
person.setFirstname("Dave");       //(2)                   
Example<Person> example = Example.of(person); //(3) 
```

1) 创建一个领域对象的新实例
2) 设置查询的属性
3) 创建```Example```


Examples are ideally be executed with repositories. To do so, let your repository interface extend QueryByExampleExecutor<T>. The following listing shows an excerpt from the QueryByExampleExecutor interface:
理想情况下，```Example```是使用存储库执行的。为此，让存储库接口扩展```QueryByExampleExecutor<T>```。以下列表显示了```QueryByExampleExecutor```接口的摘录：

```java
public interface QueryByExampleExecutor<T> {
  <S extends T> S findOne(Example<S> example);
  <S extends T> Iterable<S> findAll(Example<S> example);
  // … more functionality omitted.
}
```

### 5.6.3 Example Matcher

示例不限于默认设置。可以使用```ExampleMatcher```为字符串匹配、空处理和特定于属性的设置指定自己的默认值，如下例所示：

```java
Person person = new Person();     //(1)                      
person.setFirstname("Dave");          //(2)                 

ExampleMatcher matcher = ExampleMatcher.matching() //(3)     
  .withIgnorePaths("lastname")                     //(4)    
  .withIncludeNullValues()                         //(5)
  .withStringMatcherEnding();                      //(6)    

Example<Person> example = Example.of(person, matcher); //(7)
```

1) 创建领域对象的新实例.
2) 设置属性.
3) 创建一个 ```ExampleMatcher```用于匹配所有值,就算没有进一步的配置,这个阶段也可以进行使用. to expect all values to match. It is usable at this stage even without further configuration.
Construct a new ExampleMatcher to ignore the lastname property path.
Construct a new ExampleMatcher to ignore the lastname property path and to include null values.
Construct a new ExampleMatcher to ignore the lastname property path, to include null values, and to perform suffix string matching.
Create a new Example based on the domain object and the configured ExampleMatcher.



