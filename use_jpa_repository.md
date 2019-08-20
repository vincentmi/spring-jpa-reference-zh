# 4 使用Spring Data 仓库

Spring Data 仓库抽象层的目标是为了显著的减少进行数据持久层访问的代码量.


## 4.1. 核心概念

Spring Data 仓库最重要的抽象接口是```Repository```,他使用领域类以及领域类的ID作为参数进行管理.这个接口主要扮演一个标记接口.用来捕捉要使用的类型和帮组发现扩展自该接口的接口.```CrudRepository``` 为管理的实体类提供复杂的CRUD功能.


Example 3. CrudRepository 接口

```java
public interface CrudRepository<T, ID extends Serializable>
  extends Repository<T, ID> {
  <S extends T> S save(S entity);     //1   
  Optional<T> findById(ID primaryKey);     //2 
  Iterable<T> findAll();        //3       
  long count();                    //4    
  void delete(T entity);              //5 
  boolean existsById(ID primaryKey);   //6
  // … more functionality omitted.
}
```

1) 保存这个实体 

2) 使用给定的ID返回实体 

3) 返回所有实体 

4) 返回实体的个数 

5) 删除给定的实体 

6) 检查指定ID的实体是否存在 


> 我们还提供基于持久技术特性的一些抽象,比如``` JpaRepository```,或者 ```MongoRepository```.这些接口继承了```CrudRepository```的一些功能,并暴露一些基于持久层的不太通用的额外功能.
> 

在 CrudRepository 之上我们有一个```PagingAndSortingRepository```抽象层来实现对实体的分页访问.

Example 4. PagingAndSortingRepository 接口

```java
public interface PagingAndSortingRepository<T, ID extends Serializable>
  extends CrudRepository<T, ID> {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```
每页20个,要访问用户实体的第二页,可以这样做:

```java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Page<User> users = repository.findAll(PageRequest.of(1, 20));
```

除了```query```方法之外，还可以使用```count```和```delete```方法。 下面显示了```Count```语句的定义

Example 5. 定义count查询

```java
interface UserRepository extends CrudRepository<User, Long> {

  long countByLastname(String lastname);
}
```

下面是```delete```方法的定义

```java
interface UserRepository extends CrudRepository<User, Long> {

  long deleteByLastname(String lastname);

  List<User> removeByLastname(String lastname);
}
```


## 4.2  Query方法

标准的CRUD 操作经常对数据库底层进行查询.使用Spring Data 使用以下四步进行定义:

#### 1.定义一个扩展自```Repository```或者他的子接口的接口.并指定实体的类和ID的类型
 如下
 
 ```java
 interface PersonRepository extends Repository<Person, Long> { … }
 ```
#### 2.在接口中添加Query方法

```java
 interface PersonRepository extends Repository<Person, Long> {
  List<Person> findByLastname(String lastname);
}
```

#### 3 设置Spring为这些接口创建代理实例.可以通过```JavaConfig```或者```XML```来做:

##### 3.1 使用JavaConfig 
    
```java
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@EnableJpaRepositories
class Config {}
```
##### 3.2 使用XML配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:jpa="http://www.springframework.org/schema/data/jpa"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
     https://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/data/jpa
     https://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

   <jpa:repositories base-package="com.acme.repositories"/>

</beans>
```

这个例子中使用了```JPA```的命名空间.如果你使用了另外的存储,你需要将命名空间换成你使用的模块.你需要把jpa换成你要使用的,比如 ```mongodb```.

另请注意，JavaConfig变量未显示的配置包名，因为默认情况下会使用被注解的类的包。 要自定义要扫描的包，请使用特定于数据存储库的 ```@Enable${store}Repositories```注解的```basePackage``` 属性。


#### 4.注入仓库的实例并使用它. 示例如下:

```java
class SomeClient {

  private final PersonRepository repository;

  SomeClient(PersonRepository repository) {
    this.repository = repository;
  }

  void doSomething() {
    List<Person> persons = repository.findByLastname("Matthews");
  }
}
```

## 4.3 定义仓库接口

首先,定义指定领域类的仓库接口,接口必须继承```Repository```并设置类型参数为领域类和领域类的ID.如果你想暴露CRUD方法,可以用 ```CrudRepository```来代替```Repository```.


### 4.3.1. 调整仓库的定义

通常，存储仓库接口会扩展 ```Repository```，```CrudRepository```或```PagingAndSortingRepository```。 或者，如果您不想扩展Spring Data接口，还可以使用```@RepositoryDefinition``` 进行注解扩展```CrudRepository```暴露的一整套操作实体的方法。 如果您选择Spring实现的方法，请将要从CrudRepository公开的方法复制到域存储库中。

下面的例子展示了如何选择暴露CRUD方法(在这个例子中是 findByID和 save:

```java
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {

  Optional<T> findById(ID id);

  <S extends T> S save(S entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```

上面的例子,你定义了一个基础的接口用于你的所以仓库,保留了```findById``` 和 ```Save``` 方法.这些方法会路由到你选择的 Spring Data的实现类(例如,如果你使用JPA,他的实现是 ```SimpleJpaRepository```),因为他们与```CrudRepository```的方法签名匹配,所以 ```UserRepository```现在可以```save```和```findById```,并且可以通过email查询用户.

> 中间接口使用 ```@NoRepositoryBean``` 进行注解,这样Spring Data 就不会为他创建运行时的实例.
> 

### 4.3.2. Null Handling of Repository Methods

从Spring Data 2.0开始,仓库的CRUD方法返回了Java8的 ```Optional```对象来检查是否存在值.Spring Data 支持在Query方法中返回如下类型的包装类:

```com.google.common.base.Optional```

```scala.Option```

```io.vavr.control.Option```

```javaslang.control.Option (deprecated as Javaslang is deprecated)```

或者Query方法也可以选择不使用包装类,如果查询的结果不存在则返回```null```.
保证返回集合，集合替代，包装器和stream的存储库方法永远不会返回null，而是返回相应的空表示。 有关详细信息，请参阅[“存储库查询返回类型”](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#repository-query-return-types)。

#### Nullability注解

您可以使用Spring Framework的可空性注释来表达存储库方法的可空性约束。 它们在运行时提供了一种工具友好的方法和opt-in null检查，如下所示：

```@NonNullApi```：在包级别上使用，以声明参数和返回值的默认行为是不接受或生成空值。

```@NonNull```：用于参数或返回值，该值不能为null（参数和@NonNullApi适用的返回值不需要）。

```@Nullable```：用于可以为null的参数或返回值。


Spring注释是使用JSR 305注释进行元注释的。 JSR 305元注释允许IDEA，Eclipse和Kotlin等工具供应商以通用方式提供空安全支持，而无需对Spring注释进行硬编码支持。 要为查询方法启用运行时检查可空性约束，需要在package-info.java中使用Spring的@NonNullApi来激活包级别的非可空性，如以下示例所示：

```java
@org.springframework.lang.NonNullApi
package com.acme;
```

一旦存在非null默认值，就会在运行时验证存储库查询方法调用的可空性约束。 如果查询执行结果违反了定义的约束，则抛出异常。 当方法返回null但声明为非可空（默认情况下，在存储库所在的包中定义了注释）时会发生这种情况。 如果您想再次选择可以为空的结果，请在各个方法上有选择地使用@Nullable。 使用本节开头提到的结果包装器类型将继续按预期工作：空结果将转换为表示缺席的值。


```java
package com.acme;                                    //(1)                   
import org.springframework.lang.Nullable;
interface UserRepository extends Repository<User, Long> {
  User getByEmailAddress(EmailAddress emailAddress);        //(2)             
  @Nullable
  User findByEmailAddress(@Nullable EmailAddress emailAdress);      //(3)        
  Optional<User> findOptionalByEmailAddress(EmailAddress emailAddress);    //(4)
}
```

(1) 进行非空检测行为的包

(2) 当查询的结果为空时抛出```EmptyResultDataAccessException``` 异常,当传递给方法的参数```emailAddress```为空的时候抛出 ```IllegalArgumentException```异常.

(3) 当查询的结果为空的时候返回```null```,同时也允许参数 ```emailAddress```为```null```

(4) 当查询的结果为空的时候回返回 ```Optional.empty()```,当传递给```emailAddress```的值为空是抛出```IllegalArgumentException```异常.  

Nullability in Kotlin-based Repositories
#### Kotlin的空值约束

Kotlin对语言中的可空性约束进行了定义。 Kotlin代码编译为字节码，它不通过方法签名表达可空性约束，而是通过编译元数据表达。 确保在项目中包含kotlin-reflect 包，以便对Kotlin的可空性约束进行内部检查。 Spring Data库使用语言机制来定义这些约束以应用相同的运行时检查，如下所示：

示例10.对Kotlin存储库使用可空性约束

```kotlin
interface UserRepository : Repository<User, String> {

  fun findByUsername(username: String): User     //(1) 

  fun findByFirstname(firstname: String?): User?  //(2)
}
```

(1) 这个方法被kotlin默认定义为方法参数和返回值都不为空.Kotlin编译器拒绝在方法中传递```null```.如果查询执行结果产生了一个空的结果,将会抛出```EmptyResultDataAccessException```异常.
(2)  这个方法接受  ```null```作为 ```firstname```参数 的值,如果查询结果为空或直接返回 ```null```


#### 4.3.3. 在仓库中使用多个Spring  Data模块

在应用中使用唯一的Spring Data模块较为简单.因为所有接口都绑定到Spring Data模块.有时候应用程序需要使用多个Spring Data模块.在这种情况下仓库定义必须区分持久层的技术.当Spring在类路径上检测到多个库的工厂时,Spring Data 将会进入到严格配置模式.严格模式使用库或者领域类的详细信息来决定存储库与Spring Data 模块的绑定:

> 如果库的定义扩展了模块特定的库，那么它是特定Spring Data 模块的有效候选者。

> 如果领域类有Spring Data模块的专有注解,那么他是特定Spring Data模块的有效候选者,比如 JPA的 ```@Entity```注解.比如 ```@Document```注解用于Spring Data MongoDB和Spring Data Elasticsearch.

下面的例子展示了使用模块相关接口的例子: (例子中使用JPA)

示例 11. 仓库定义使用了模块指定的接口

```java
interface MyRepository extends JpaRepository<User, Long> { }

@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends JpaRepository<T, ID> {
  …
}

interface UserRepository extends MyBaseRepository<User, Long> {
  …
}
```

```MyRepository``` 和 ```UserRepository```继承自 ```JpaRepository```. 他们是Spring Data JPA 模块的候选者.


示例 12. 仓库定义使用通用接口

```java
interface AmbiguousRepository extends Repository<User, Long> {
 …
}

@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends CrudRepository<T, ID> {
  …
}

interface AmbiguousUserRepository extends MyBaseRepository<User, Long> {
  …
}

```
```AmbiguousRepository``` 和 ```AmbiguousUserRepository``` 继承自 Repository 和 CrudRepository .单个数据模块没有问题.但是多个模块的时候就不能确定应该绑定到哪个模块.
 
下面的示例演示了使用注解的领域类:

示例 13. 仓库使用领域逻辑注解

```java
interface PersonRepository extends Repository<Person, Long> {
 …
}
@Entity
class Person {
  …
}
interface UserRepository extends Repository<User, Long> {
 …
}
@Document
class User {
  …
}
```
```PersonRepository```引用了```Person```, ```Person```通过 JPA的 ```@Entity```进行注解.所以很明显这个仓库属于Spring Data JPA.```UserRepository```引用了```User```,而```User```使用了属于MongoDB的```@Document```注解.

下面是一个错误的示例,领域逻辑使用了混合的注解:

示例 14. 仓库使用了混合注解的领域类

```java
interface JpaPersonRepository extends Repository<Person, Long> {
 …
}
interface MongoDBPersonRepository extends Repository<Person, Long> {
 …
}

@Entity
@Document
class Person {
  …
}
```
这个例子中一个领域类同时使用了JPA和Spring Data MongoDB 的注解.他定义了两个仓库 ```JpaPersonRepository```和```MongoDBPersonRepository```.希望一个用于```JPA```另一个用于```MongoDB```,Spring Data 不再能区分存储库.这将导致未定义的行为.

库类型定义和领域类注解用于准确的存储库配置，用于识别特定Spring Data 模块的存储库候选。 在同一域类型上使用多个持久性技术特定的注释是可能的，并允许跨多种持久性技术重用域类型。 但是，Spring Data不再能够确定用于绑定存储库的唯一模块。

区分存储库的最后一种方法是通过确定存储库基础包的范围。 基础包定义了扫描存储库接口定义的起点，这意味着将存储库定义放在相应的包中。 默认情况下，注释驱动的配置使用配置类的包。 基于XML的配置中的基础包是必需的。

示例 15. 在注解中配置基础包

```java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
interface Configuration { }
```


## 4.4 定义查询方法

库的代理有两种方式通过类方法来定义查询 1) 直接使用方法名  2) 使用手动定义的查询语句

可用的方法取决于具体的存储.但是必须有一个策略创建实际的查询内容.下一节将介绍可用选项.


### 4.4.1 查询查找策略

存储库基础架构可用使用以下的策略来解析查询.使用XML配置你可以通过```query-lookup-strategy```属性在命名空间来配置策略.使用Java配置你可以使用```Enable${store}Repositories```注解的```queryLookupStrategy```属性来进行配置.一些数据存储可能不支持某些策略.

-```CREATE``` 尝试从查询方法名称构造特定存储的查询.通常是删除方法的前缀并解析方法的余下的部分.你可以从后续的内容找到详情
- ```USE_DECLARED_QUERY``` 尝试查找一个已经定义的查询语句,如果找不到会抛出异常.查询可以通过注解来声明,也可以通过其他方式.如果仓库的基础架构在引导时未找到这个已经定义的查询语句将会导致一个失败.
- ```CREATE_IF_NOT_FOUND``` (默认) 结合了 ```CREATE``` 和 ```USE_DECLARED_QUERY```. 首先查询是不是有已经定义的查询语句.如果没有找到,则会创建根据方法名解析的查询语句.这是默认的查找策略，因此，如果您未明确配置任何内容，则使用此策略。 它允许通过方法名称进行快速查询定义，还可以根据需要引入声明的查询来自定义这些查询。

### 4.4.2 创建查询语句

Spring Data 仓库基础架构的查询构造器对于创建对实体的基本查询非常有效.这个机制剥离方法中的``` find…By```, ```read…By```, ```query…By```, ```count…By```, 和 ```get…By ```解析剩下的部分.在开始的部分可以包含更多表达式,比如使用```Distinct```来为查询语句设置一个 ```distinct```标志.第一个```By```扮演一个分隔符来界定实际条件的开始.最基本的使用方式,你可以使用实体的属性用```And```或者```Or```连接起来.一下示例展示了如何创建一些查询: 

示例:16 使用方法名创建查询

```java
interface PersonRepository extends Repository<User, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}
```

解析的实际结果取决于你正在使用的持久化层.但是有一些一般需要注意的事项:

- 表达式通常是由可遍历的属性和运算符连接起来.你可以使用```AND```和```OR```将他们连接起来.你也可以使用其他的运算符,比如 ```Between``` , ```LessThan```,```GreaterThan```,```Like``` 来连接表达式.支持的运算符与数据库有关,请参阅文档的相关部分
- 方法解析器支持为各个属性设置```IgnoreCase```标志（例如，```findByLastnameIgnoreCase（...）```）或支持忽略大小写的类型的所有属性（通常是字符串类型 - 例如，```findByLastnameAndFirstnameAllIgnoreCase（...）```）。 是否支持忽略大小写可能因存储而异，因此请参阅参考文档中有关特定于商店的查询方法的相关章节。
- 你也可以通过加入```OrderBy```到查询方法来进行静态排序,并使用 ```Asc```或者```Desc```来提供排序的方向.要创建动态排序查询后续章节(4.4.4).

### 4.4.3. 属性表达式

如之前的示例所示,属性表达式只能引用被托管的实体的直接属性.你需要确定在创建查询时被解析的属性已经是被托管的领域类的属性.但是，您也可以通过遍历嵌套属性来定义约束。 请考虑以下方法定义：

```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```

假设一个```Person```有一个 包含```ZipCode```的```Address```属性.这种情况下,该方法创建属性遍历 ```x.address.zipCode```.解析算法首先将整个部分```AddressZipCode```作为属性解析,检查域类是否存在该属性(未大写的).如果存在则会使用该属性.如果不存在算法会根据驼峰从右侧截开驼峰的字串,分成头部和尾部.在我们的示例中拆分为```AddressZip```和```Code```.如果算法找到了头部字串的属性,则会使用尾部继续构建树.以同样的方法来分开尾部.如果第一次分割不匹配,算法就会从左侧开始切分,在我们的示例中会切分为```Address```和```ZipCode```并继续匹配.

虽然这应该适用于大多数情况，但算法可能会选择错误的属性。 假设```Person```类也有一个```addressZip```属性。 算法将在第一个拆分轮中匹配，选择错误的属性，并失败（因为```addressZip```的类型可能没有```Code```属性）。

要解决这种歧义，可以在方法名称中使用```_```来手动定义遍历点。 所以我们的方法名称如下：

```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```
因为我们将下划线字符视为保留字符，所以我们强烈建议遵循标准Java命名约定（即，不在属性名称中使用下划线，而是使用驼峰命名）。


### 4.4.4. 处理特殊参数



要处理查询中的参数，请定义方法参数，如前面示例中所示。 除此之外，基础架构还可识别某些特定类型（如```Pageable```和```Sort```），用于动态地对查询进行分页和排序。 以下示例演示了这些功能：

示例 17. 在方法中使用 ```Pageable```, ```Slice```, 和 ```Sort``` 

```java
Page<User> findByLastname(String lastname, Pageable pageable);
Slice<User> findByLastname(String lastname, Pageable pageable);
List<User> findByLastname(String lastname, Sort sort);
List<User> findByLastname(String lastname, Pageable pageable);
```

第一个方法让你可以传递一个 ```org.springframework.data.domain.Pageable```实例给查询方法用于动态的添加分页到你定义的查询.一个分页需要知道元素的总数和页数.他通过底层架构触发底层的一个计数查询来计算总数.这个操作可能比较昂贵(取决于所使用的存储). 你可以返回一个```Slice```来替换.```Slice``` 只需要知道下个```Slice```是否可用,这在遍历大结果集的时候可能就足够了.


排序选项也通过Pageable实例处理。 如果只需要排序，请在方法中添加```org.springframework.data.domain.Sort```参数。 如您所见，也可以返回```List```。 在这种情况下，不会创建构建实际页面实例所需的其他元数据（反过来，这意味着不会发出必要的附加计数查询）。 相反，它限制查询仅查找给定范围的实体。

> 要了解整个查询的页数，必须触发其他计数查询。 默认情况下，此查询是从您实际触发的查询派生的。
> 


### 4.4.5.限制查询结果

查询方法的返回结果可以使用```first```和```top```关键字进行限制.这些关键字可以互换使用.可选的数字可以附加到```top```或者```first```以指定返回的结果大小.如歌省略数字,则假定结果大小为1,一下示例显示如何限制大小.

示例 18. 使用```Top``` 和 ```First```限制结果大小

```java
User findFirstByOrderByLastnameAsc();
User findTopByOrderByAgeDesc();
Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);
Slice<User> findTop3ByLastname(String lastname, Pageable pageable);
List<User> findFirst10ByLastname(String lastname, Sort sort);
List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

限制表达式也支持```Distinct```关键字.另外,对于将结果限制为一个实例的查询,支持使用```Optional```对结果进行包装.

如果将分页或切片应用于限制查询分页（以及可用页数的计算），则将其应用于限定的结果中。

>通过使用Sort参数将结果与动态排序结合使用，可以表达“K”最小元素和“K”元素的查询方法。
>

4.4.6. 查询结果流式处理

查询方法可以通过使用Java 8的 ```Stream<T>```作为返回类型来进行增量处理.而不是将查询结果包装在```Stream```中，数据存储相关的方法用于执行流式处理，如以下示例所示：

示例 19. 使用 Java 8 ```Stream<T>```对结果进行流式处理

```java
@Query("select u from User u")
Stream<User> findAllByCustomQueryAndStream();

Stream<User> readAllByFirstnameNotNull();

@Query("select u from User u")
Stream<User> streamAllPaged(Pageable pageable);
```

Stream可能会包装底层数据存储相关的资源，因此必须在使用后关闭。 您可以使用close（）方法或使用Java 7 try-with-resources块手动关闭Stream，如以下示例所示：

示例 20. 使用```try-with-resources```区块操作 ```Stream<T>``` 结果

```java
try (Stream<User> stream = repository.findAllByCustomQueryAndStream()) {
  stream.forEach(…);
}
```

>不是所有的Spring Data 模块当前都支持 ```Stream<T>```作为返回类型.


### 4.4.7. 异步查询结果

可以使用Spring的异步方法执行功能异步运行存储库查询。 这意味着该方法在调用时立即返回，而实际的查询执行发生在已提交给Spring TaskExecutor的任务中。 异步查询执行与反应式查询执行不同，不应混合使用。 有关反应支持的更多详细信息，请参阅特定存储的文档。 以下示例显示了许多异步查询：

```java
@Async
Future<User> findByFirstname(String firstname);     //(1)          

@Async
CompletableFuture<User> findOneByFirstname(String firstname); //(2)

@Async
ListenableFuture<User> findOneByLastname(String lastname);   //(3) 
```

(1) 使用 ```java.util.concurrent.Future ```作为返回类型.

(2) 使用 Java 8 ```java.util.concurrent.CompletableFuture```作为返回类型.

(3) 使用 ```org.springframework.util.concurrent.ListenableFuture```作为返回类型.

## 4.5. 创建仓库实例

在本节中，您将为定义的存储库接口创建实例和bean定义。 一种方法是使用随每个支持存储库机制的Spring Data模块一起提供的Spring命名空间，尽管我们通常建议使用Java配置。

### 4.5.1. XML配置

每个Spring Data模块都包含一个 ```repositories```元素让你定义一个基础包让Spring进行扫描.如下示例:

示例 21.通过XML启用Spring Data 仓库

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns:beans="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://www.springframework.org/schema/data/jpa"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/data/jpa
    https://www.springframework.org/schema/data/jpa/spring-jpa.xsd">
  <repositories base-package="com.acme.repositories" />
</beans:beans>
```

在之前的示例中,指示Spring扫描```com.acme.repositories```和他的子包.对找到的所以接口底层架构注册与持久层相关的```FactoryBean```,以创建处理查询方法相关的代理.每个Bean都是从接口名称下注册的.因此```UserRepository```的接口将在```userRepository```下注册.base-package属性允许使用通配符以便您可以定义扫描包的模式.

#### 使用过滤器

默认情况下，基础结构会选择扩展位于已配置的基础包下的特定于持久性技术的Repository子接口的每个接口，并为其创建一个bean实例。 但是，您可能希望对哪些接口为其创建bean实例进行更细粒度的控制。为此，请在```<repositories />```元素中使用```<include-filter />```和```<exclude-filter />```元素。 语义完全等同于Spring的上下文命名空间中的元素。 有关详细信息，请参阅这些元素的Spring参考文档。

例如，要将某些接口从实例化中排除为存储库bean，可以使用以下配置：

示例 22. 使用 ```exclude-filter``` 元素

```xml
<repositories base-package="com.acme.repositories">
  <context:exclude-filter type="regex" expression=".*SomeRepository" />
</repositories>
```

前面的示例排除了以```SomeRepository```结束的所有接口的实例化。

## 4.5.2 Java配置

还可以通过在```JavaConfig```类上使用特定于存储的```@ Enable${store}Repositories```注解来触发存储库基础结构。 有关Spring容器的基于Java的配置的介绍，请参阅[相关文档](https://docs.spring.io/spring/docs/5.1.9.RELEASE/spring-framework-reference/core.html#beans-java)。

启用Spring Data存储库的示例配置类似于以下内容：
示例23.基于样本注释的存储库配置

```java
@Configuration
@EnableJpaRepositories("com.acme.repositories")
class ApplicationConfiguration {

  @Bean
  EntityManagerFactory entityManagerFactory() {
    // …
  }
}
```

上面的示例使用JPA特定的注释，您可以根据实际使用的存储模块进行更改。 这同样适用于```EntityManagerFactory``` bean的定义。 请参阅涵盖特定于存储的配置的部分。


### 4.5.3 独立使用

您还可以在Spring容器之外使用存储库基础架构 - 例如，在CDI环境中。 您仍然需要在类路径中使用一些Spring库，但通常也可以通过编程方式设置存储库。 提供存储库支持的Spring Data模块提供了一个特定于持久性技术的```RepositoryFactory```，您可以按如下方式使用

示例24.存储库工厂的独立使用

```java
RepositoryFactorySupport factory = … // Instantiate factory here
UserRepository repository = factory.getRepository(UserRepository.class);
```

## 4.6. 定制Spring Data 仓库的实现

本节介绍存储库自定义以及片段如何构成复合存储库。

当查询方法需要不同的行为或无法通过查询派生实现时，则需要提供自定义实现。 Spring Data存储库允许您提供自定义存储库代码，并将其与通用CRUD抽象和查询方法功能集成。

### 4.6.1. 自定义单独存储库

要使用自定义功能丰富存储库，必须首先定义片段接口和自定义功能的实现，如以下示例所示：

示例25.自定义存储库功能的接口

```java
interface CustomizedUserRepository {
  void someCustomMethod(User user);
}
```

然后，您可以让您的存储库接口从片段接口进一步扩展，如以下示例所示：

示例26.自定义存储库功能的实现

```java
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {

  public void someCustomMethod(User user) {
    // Your custom implementation
  }
}
```

> 与片段接口对应的类名最重要的部分是```Impl```后缀。

实现本身不依赖于Spring Data，可以是常规的Spring bean。 因此，您可以使用标准依赖项注入行为来注入对其他bean（例如```JdbcTemplate```）的引用，等等。

您可以让存储库接口扩展片段接口，如以下示例所示：

示例27.对存储库界面的更改

```java
interface UserRepository extends CrudRepository<User, Long>, CustomizedUserRepository {

  // Declare query methods here
}
```

使用存储库接口扩展片段接口可以组合CRUD和自定义功能，并使其可供客户端使用。

Spring Data存储库通过使用形成存储库组合的片段来实现。片段是基础仓库,切面(比如 ```QueryDSL```) 和自定义接口的实现.每次向存储库界面添加接口时都可以通过添加片段来增强组合.每个Spring Data模块都提供了基本存储库和存储库切面的实现.

以下示例显示了自定义接口及其实现：

示例28.具有其实现的片段

```java
interface HumanRepository {
  void someHumanMethod(User user);
}

class HumanRepositoryImpl implements HumanRepository {
  public void someHumanMethod(User user) {
    // Your custom implementation
  }
}

interface ContactRepository {
  void someContactMethod(User user);
  User anotherContactMethod(User user);
}

class ContactRepositoryImpl implements ContactRepository {
  public void someContactMethod(User user) {
    // Your custom implementation
  }

  public User anotherContactMethod(User user) {
    // Your custom implementation
  }
}
```

以下示例显示了扩展```CrudRepository```的自定义存储库的接口：

示例29.对存储库接口的更改

```java
interface UserRepository extends CrudRepository<User, Long>, HumanRepository, ContactRepository {

  // Declare query methods here
}
```

存储库可以由多个自定义实现组成，这些实现按其声明的顺序导入。 自定义实现的优先级高于基本实现和存储库方面。 如果两个片段提供相同的方法签名，则此排序允许您覆盖基本存储库和方面方法并解决歧义。 存储库片段不限于在单个存储库接口中使用。 多个存储库可以使用片段接口，允许您跨不同的存储库重用自定义。

以下示例显示了存储库片段及其实现：

示例30.覆盖```save()```方法的片段

```java
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}

class CustomizedSaveImpl<T> implements CustomizedSave<T> {

  public <S extends T> S save(S entity) {
    // Your custom implementation
  }
}
```

以下示例显示了使用前面的存储库片段的存储库：
示例31.定制的存储库接口

```java


interface UserRepository extends CrudRepository<User, Long>, CustomizedSave<User> {
}

interface PersonRepository extends CrudRepository<Person, Long>, CustomizedSave<Person> {
}
```

#### 配置

如果使用命名空间配置，则存储库基础结构会尝试通过扫描其找到存储库的包下面的类来自动检测自定义实现片段。 这些类需要遵循将命名空间元素的```repository-impl-postfix```属性附加到片段接口名称的命名约定。 此后缀默认为```Impl```。 以下示例显示了使用默认后缀的存储库以及为后缀设置自定义值的存储库：

示例 32. 配置示例

```xml
<repositories base-package="com.acme.repository" />
<repositories base-package="com.acme.repository" repository-impl-postfix="MyPostfix" />
```

第一个配置查找 ```com.acme.repository.CustomizedUserRepositoryImpl```来作为一个自定义存储的实现.第二个则会查询 ```com.acme.repository.CustomizedUserRepositoryMyPostfix```

#### 解决歧义

如果在不同的包中找到具有匹配类名的多个实现，则Spring Data使用bean名称来标识要使用的名称。

为之前的```CustomizedUserRepository```在下面给出了两个自定义实现，则会使用第一个实现。 它的bean名称是```customizedUserRepositoryImpl```，它与片段接口（```CustomizedUserRepository```）和后缀```Impl```的名称相匹配。

示例33.歧义的解决方案

```java
package com.acme.impl.one;
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {
  // Your custom implementation
}
```

```java
package com.acme.impl.two;
@Component("specialCustomImpl")
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {
  // Your custom implementation
}
```

如果使用```@Component（“specialCustom”）```注释```UserRepository```接口，则bean名称加上Impl将匹配为```com.acme.impl.two```中的存储库实现定义的名称，并使用它而不是第一个。


#### 手工装配

如果您的自定义实现仅使用基于注释的配置和自动装配，则前面的方法效果很好，因为它被视为和其他Spring bean一样.如果您的实现片段bean需要手工装配，您可以声明bean并根据前一节中描述的约定命名它。 然后，基础架构按名称引用手动定义的bean定义，而不是自己创建一个。 以下示例显示如何手动连接自定义实现：

示例:手动装配自定义实现

```xml
<repositories base-package="com.acme.repository" />
<beans:bean id="userRepositoryImpl" class="…">
  <!-- further configuration -->
</beans:bean>
```

###4.6.2. 自定义基础库

当你需要自定义基本存储库的行为以便所有存储库都收到影响,采用上一节的方式需要为每个存储库添加自定义接口.要更改所有库的行为,可以继承基于特定持久化库的基础库的实现.然后，此类充当存储库代理的自定义基类，如以下示例所示：

示例35.自定义存储库基类

```java


class MyRepositoryImpl<T, ID extends Serializable>
  extends SimpleJpaRepository<T, ID> {

  private final EntityManager entityManager;

  MyRepositoryImpl(JpaEntityInformation entityInformation,
                          EntityManager entityManager) {
    super(entityInformation, entityManager);

    // Keep the EntityManager around to used from the newly introduced methods.
    this.entityManager = entityManager;
  }

  @Transactional
  public <S extends T> S save(S entity) {
    // implementation goes here
  }
}

```

>该类需要具有特定于商店的存储库工厂实现所使用的超类的构造函数。 如果存储库基类具有多个构造函数，则覆盖采用```EntityInformation```和特定于存储的基础结构对象（例如```EntityManager```或模板类）的构造函数。

最后一步是让Spring Data基础架构了解自定义库基础类.在Java配置中你可以使用 之前示例中的```@Enable${store}Repositories```注解的 ```repositoryBaseClass```属性.

示例36.使用JavaConfig配置自定义存储库基类

```java
@Configuration
@EnableJpaRepositories(repositoryBaseClass = MyRepositoryImpl.class)
class ApplicationConfiguration { … }
```

XML命名空间中提供了相应的属性，如以下示例所示：

```xml
<repositories base-package="com.acme.repository"
     base-class="….MyRepositoryImpl" />
```

## 4.7  从聚合根发布事件

通过仓库管理的实例是是聚合的根.在领域驱动设计的应用中.聚合根通常会发布领域事件.Spring Data提交一个叫做```@DomainEvents```的注解,你可以在聚合根的方法上使用该注解.以使事件的发布尽可能的简单.如下面的示例所示:

示例38.从聚合根发布领域事件

```java
class AnAggregateRoot {

    @DomainEvents  //(1)
    Collection<Object> domainEvents() {
        // … return events you want to get published here
    }

    @AfterDomainEventPublication //(2) 
    void callbackMethod() {
       // … potentially clean up domain events list
    }
}
```

(1) 这个方法使用```@DomainEvents```可以返回一个单独的事件实例或者一组.该方法不能包含参数

(2) 当所有事件发布之后,我们有一个包含  ```@AfterDomainEventPublication```注解的方法. 它可用于潜在地清除要发布的事件列表（以及其他用途）。

每次调用存储库的```save()```方法时都会调用 以上方法.


## 4.8. Spring Data 扩展

本节介绍了一组Spring Data扩展，它们可以在各种上下文中使用Spring Data。 目前，大多数集成都针对Spring MVC

###   4.8.1. Querydsl 扩展 

[Querydsl](https://www.querydsl.com/)是一个框架，可以通过其流畅的API构建静态类型的SQL类查询。

几个Spring Data模块通过```QuerydslPredicateExecutor```提供与Querydsl的集成，如以下示例所示：

例39. QuerydslPredicateExecutor接口

```java
public interface QuerydslPredicateExecutor<T> {

  Optional<T> findById(Predicate predicate);   //(1)

  Iterable<T> findAll(Predicate predicate);   //(2)

  long count(Predicate predicate);            //(3)

  boolean exists(Predicate predicate);        //(4)

  // … more functionality omitted.
}
```

(1) 查找并返回与```Predicate```匹配的单个实体

(2) 查找并返回与```Predicate```匹配的所有实体

(3) 返回与```Predicate```匹配的实体个数.

(4) 返回与```Predicse```匹配的实体是否存在. 

要使用```Querydsl```支持，请在存储库接口上扩展```QuerydslPredicateExecutor```，如以下示例所示:

示例40.对存储库的Querydsl集成

```java
interface UserRepository extends CrudRepository<User, Long>, QuerydslPredicateExecutor<User> {
}
```

前面的示例允许您使用Querydsl ```Predicate```实例编写类型安全查询，如以下示例所示：

```java
Predicate predicate = user.firstname.equalsIgnoreCase("dave")
	.and(user.lastname.startsWithIgnoreCase("mathews"));
userRepository.findAll(predicate);
```

### 4.8.2. Web支持


>该部分包含Spring Data Web支持的文档，因为它是在Spring Data Commons的当前（及更高版本）版本中实现的。 由于新引入的支持更改了许多内容，因此我们在[web.legacy]中保留了以前行为的文档。
>
>

支持存储库编程模型的Spring Data模块具有各种Web支持。 Web相关组件要求Spring MVC JAR位于类路径上。 其中一些甚至提供与Spring HATEOAS的集成。 通常，通过在JavaConfig配置类中使用```@EnableSpringDataWebSupport```批注来启用集成支持，如以下示例所示：

```java
@Configuration
@EnableWebMvc
@EnableSpringDataWebSupport
class WebConfiguration {}
```

```@EnableSpringDataWebSupport```注释注册了一些我们将稍微讨论的组件。 它还将检测类路径上的Spring HATEOAS，并为它注册集成组件（如果存在）。

或者，如果使用XML配置，请将```SpringDataWebConfiguration```或```HateoasAwareSpringDataWebConfiguration```注册为Spring bean，如以下示例所示（对于```SpringDataWebConfiguration```）：

```xml


<bean class="org.springframework.data.web.config.SpringDataWebConfiguration" />

<!-- If you use Spring HATEOAS, register this one *instead* of the former -->
<bean class="org.springframework.data.web.config.HateoasAwareSpringDataWebConfiguration" />

```

#### 基本Web支持 

上一节中显示的配置注册了一些基本组件：

- 一个```DomainClassConverter```让Spring MVC从请求参数或路径变量中解析存储库管理的域类的实例。
- ```HandlerMethodArgumentResolver```实现让Spring MVC从请求参数中解析```Pageable```和```Sort```实例。

#### DomainClassConverter

```DomainClassConverter```允许您直接在Spring MVC控制器方法签名中使用域类型，因此您无需通过存储库手动查找实例，如以下示例所示：

```java
@Controller
@RequestMapping("/users")
class UserController {

  @RequestMapping("/{id}")
  String showUserForm(@PathVariable("id") User user, Model model) {

    model.addAttribute("user", user);
    return "userForm";
  }
}
```

如您所见，该方法直接接收```User```实例，无需进一步查找。 可以通过让Spring MVC首先将路径变量转换为域类的```id```类型来解析实例，并最终通过在为域类型注册的存储库实例上调用```findById（...）```来访问实例。

>目前，存储库必须实现CrudRepository才有资格被发现进行转换。

####   HandlerMethodArgumentResolvers 处理 Pageable 和 Sort

上一节中显示的配置代码段还注册了```PageableHandlerMethodArgumentResolver```以及```SortHandlerMethodArgumentResolver```的实例。 注册启用```Pageable```和```Sort```作为有效的控制器方法参数，如以下示例所示：

```java


@Controller
@RequestMapping("/users")
class UserController {

  private final UserRepository repository;

  UserController(UserRepository repository) {
    this.repository = repository;
  }

  @RequestMapping
  String showUsers(Model model, Pageable pageable) {

    model.addAttribute("users", repository.findAll(pageable));
    return "users";
  }
}
```

前面的方法签名导致Spring MVC尝试使用以下默认配置从请求参数派生Pageable实例：


|page| 你要接收的页码 0 -索引,默认为 0 |
|---|---|
|size| 每页包含的数据条数  |
|sort| 排序字段 格式为(排序字段,排序顺序) ,顺序取值为 ```ASC```或者```DESC```默认 的排序为正序,如果 你要切换方向请使用多个排序参数,例如 ```?sort=firstname&sort=lastname,asc| 

要自定义此行为，请分别注册实现```PageableHandlerMethodArgumentResolverCustomizer```接口或```SortHandlerMethodArgumentResolverCustomizer```接口的Bean。 调用其```customize（）```方法，允许您更改设置，如以下示例所示：

```java
@Bean SortHandlerMethodArgumentResolverCustomizer sortCustomizer() {
    return s -> s.setPropertyDelimiter("<-->");
}
```

如果设置现有```MethodArgumentResolver```的属性不足以满足您的需要，请扩展```SpringDataWebConfiguration```或启用HATEOAS的等效项，覆盖```pageableResolver（）```或```sortResolver（）```方法，并导入自定义配置文件，而不是使用```@Enable```注释。

如果需要从请求中解析多个```Pageable```或```Sort```实例（例如，对于多个表），可以使用Spring的```@Qualifier```注释来区分彼此。 然后，请求参数必须以```${qualifier} _```为前缀。 以下示例显示了生成的方法签名：

```java
String showUsers(Model model,
      @Qualifier("thing1") Pageable first,
      @Qualifier("thing2") Pageable second) { … }
```

你必须填充```thing1_page```和 ```thing2_page```等等.

传递给方法的默认```Pageable```相当于```PageRequest.of（0,20）```，但可以通过在```Pageable```参数上使用```@PageableDefault```注释进行自定义。


#### 对```Pageables```的超媒体支持


Spring HATEOAS附带了一个表示模型类（```PagedResources```），它允许使用必要的页面元数据丰富页面实例的内容以及允许客户端轻松浏览页面的链接。 将```Page```转换为```PagedResources```是通过Spring HATEOAS ```ResourceAssembler```接口的实现完成的，该接口称为```PagedResourcesAssembler```。 以下示例显示如何将```PagedResourcesAssembler```用作控制器方法参数：

```java
@Controller
class PersonController {

  @Autowired PersonRepository repository;

  @RequestMapping(value = "/persons", method = RequestMethod.GET)
  HttpEntity<PagedResources<Person>> persons(Pageable pageable,PagedResourcesAssembler assembler) {

    Page<Person> persons = repository.findAll(pageable);
    return new ResponseEntity<>(assembler.toResources(persons), HttpStatus.OK);
  }
}
```

如上例所示启用配置，可以将PagedResourcesAssembler用作控制器方法参数。 在其上调用资源（...）具有以下效果：

- ```Page```的内容成为```PagedResources```实例的内容。
- ```PagedResources```对象获取一个附加的```PageMetadata```实例，并使用来自```Page```和底层```PageRequest```的信息填充它。
- 根据页面的状态，```PagedResources```可能会显示并附加下一个链接。 链接指向方法映射到的URI。 添加到方法的分页参数与```PageableHandlerMethodArgumentResolver```的设置相匹配，以确保稍后可以解析链接。

假设我们在数据库中有30个Person实例。 您现在可以触发请求（GET http//localhost：8080/persons）并查看类似于以下内容的输出：

```java
{ "links" : 
    [ 
    { 
        "rel" : "next",
        "href" : "http://localhost:8080/persons?page=1&size=20 
    }
  ],
  "content" : [
     … // 20 Person instances rendered here
  ],
  "pageMetadata" : {
    "size" : 20,
    "totalElements" : 30,
    "totalPages" : 2,
    "number" : 0
  }
}
```

您会看到```assembler```生成了正确的URI，并且还选择了默认配置以将参数解析为即将发出的请求的```Pageable```。 这意味着，如果更改该配置，链接将自动遵循更改。 默认情况下，```assembler```指向它所调用的控制器方法，但是可以通过交换自定义链接来自定义链接以构建分页链接，这会重载```PagedResourcesAssembler.toResource（...）```方法。

#### Web数据绑定支持

Spring数据投影可用于通过使用JSONPath表达式来绑定传入的请求有效负载（需要Jayway JsonPath或XPath表达式（需要XmlBeam），如以下示例所示：

```java


@ProjectedPayload
public interface UserPayload {
  @XBRead("//firstname")
  @JsonPath("$..firstname")
  String getFirstname();

  @XBRead("/lastname")
  @JsonPath({ "$.lastname", "$.user.lastname" })
  String getLastname();
}
```

前面示例中显示的类型可以用作Spring MVC处理程序方法参数，也可以在```RestTemplate```方法之一上使用```ParameterizedTypeReference```。 前面的方法声明将尝试在给定文档中的任何位置查找```firstname```。 ```lastname``` XML查找在传入文档的顶级执行。 其中JSON变体首先尝试顶级姓氏，但如果前者未返回值，则还尝试嵌套在用户子文档中的lastname。 这样，可以轻松地减轻源文档结构的变化，而无需客户端调用公开的方法（通常是基于类的有效负载绑定的缺点）。

如预测中所述，支持嵌套投影。 如果方法返回复杂的非接口类型，则使用```Jackson ObjectMapper``` 映射最终值。

对于Spring MVC，只要```@EnableSpringDataWebSupport```处于活动状态，就会自动注册必要的转换器，并且类路径上可以使用所需的依赖项。 要与```RestTemplate```一起使用，请手动注册```ProjectingJackson2HttpMessageConverter```（JSON）或```XmlBeamHttpMessageConverter```。

更多内容查看 [Spring Data Examples repository](https://github.com/spring-projects/spring-data-examples)终端[web projection example](https://github.com/spring-projects/spring-data-examples/tree/master/web/projection).


#### Querydsl的WEB支持

对于那些具有```QueryDSL```集成的存储，可以从请求的query字串中包含的属性产生查询。

使用以下查询字串

```
?firstname=Dave&lastname=Matthews
```

给定前面示例中的User对象，可以使用```QuerydslPredicateArgumentResolver```将查询字符串解析为以下值。

```java
QUser.user.firstname.eq("Dave").and(QUser.user.lastname.eq("Matthews"))
```

当```@EnableSpringDataWebSupport```启用时,在路径找到```Querydsl```这个功能就会被启用。

将```@QuerydslPredicate```添加到方法签名提供了一个可立即使用的```Predicate```，可以使用```QuerydslPredicateExecutor```运行。

>  通常从方法的返回类型中解析类型信息。 由于该信息不一定与域类型匹配，因此使用```QuerydslPredicate```的root属性可能是个好主意。








 

