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

Property expressions can refer only to a direct property of the managed entity, as shown in the preceding example. At query creation time, you already make sure that the parsed property is a property of the managed domain class. However, you can also define constraints by traversing nested properties. Consider the following method signature:

```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```
Assume a Person has an Address with a ZipCode. In that case, the method creates the property traversal x.address.zipCode. The resolution algorithm starts by interpreting the entire part (AddressZipCode) as the property and checks the domain class for a property with that name (uncapitalized). If the algorithm succeeds, it uses that property. If not, the algorithm splits up the source at the camel case parts from the right side into a head and a tail and tries to find the corresponding property — in our example, AddressZip and Code. If the algorithm finds a property with that head, it takes the tail and continues building the tree down from there, splitting the tail up in the way just described. If the first split does not match, the algorithm moves the split point to the left (Address, ZipCode) and continues.

Although this should work for most cases, it is possible for the algorithm to select the wrong property. Suppose the Person class has an addressZip property as well. The algorithm would match in the first split round already, choose the wrong property, and fail (as the type of addressZip probably has no code property).

To resolve this ambiguity you can use _ inside your method name to manually define traversal points. So our method name would be as follows:

```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```
Because we treat the underscore character as a reserved character, we strongly advise following standard Java naming conventions (that is, not using underscores in property names but using camel case instead).


