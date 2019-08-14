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

In addition to query methods, query derivation for both count and delete queries is available. The following list shows the interface definition for a derived count query:

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


## 4.1. Query方法

标准的CRUD 操作经常对数据库底层进行查询.使用Spring Data 使用以下四步进行定义:

* 定义一个扩展自```Repository```或者他的子接口的接口.并指定实体的类和ID的类型
 如下
 
 ```java
 interface PersonRepository extends Repository<Person, Long> { … }
 ```
 
 * 在接口中添加Query方法

 ```java
 interface PersonRepository extends Repository<Person, Long> {
  List<Person> findByLastname(String lastname);
}
```

* 设置Spring为这些接口创建代理实例.可以通过```JavaConfig```或者```XML```来做: