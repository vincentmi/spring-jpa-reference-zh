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

上面的例子,你定义了一个基础的接口用于你的所以仓库,保留了```findById``` 和 ```Save``` 方法.这些方法会路由到你选择的 Spring Data的实现类(例如,如果你使用JPA,他的实现是 ```SimpleJpaRepository```),因为他们与```CrudRepository```的方法签名匹配,所以 ```UserRepository```现在可以save和findById,并且可以通过email查询用户.

> 中间接口使用 ```@NoRepositoryBean``` 进行注解,这样Spring Data 就不会为他创建运行时的实例.
> 

### 4.3.2. Null Handling of Repository Methods

As of Spring Data 2.0, repository CRUD methods that return an individual aggregate instance use Java 8’s Optional to indicate the potential absence of a value. Besides that, Spring Data supports returning the following wrapper types on query methods:

com.google.common.base.Optional

scala.Option

io.vavr.control.Option

javaslang.control.Option (deprecated as Javaslang is deprecated)

Alternatively, query methods can choose not to use a wrapper type at all. The absence of a query result is then indicated by returning null. Repository methods returning collections, collection alternatives, wrappers, and streams are guaranteed never to return null but rather the corresponding empty representation. See “Repository query return types” for details.



