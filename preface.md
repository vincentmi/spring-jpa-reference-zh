# 前言

Spring Data JPA  提供了JAVA持久化API \(即JPA\)的库,用于简化需要访问JPA数据源的应用程序的开发

## 1. 项目资料

* 版本库 -[https://github.com/spring-projects/spring-data-jpa](https://github.com/spring-projects/spring-data-jpa)

* BUG跟踪 -[https://jira.spring.io/browse/DATAJPA](https://jira.spring.io/browse/DATAJPA)

* 发布包 -[https://repo.spring.io/libs-release](https://repo.spring.io/libs-release)

* 里程碑包 -[https://repo.spring.io/libs-milestone](https://repo.spring.io/libs-milestone)

* 快照 -[https://repo.spring.io/libs-snapshot](https://repo.spring.io/libs-snapshot)

## 2.新特性 

### 2.1. Spring Data JPA 1.11

Spring Data JPA 1.11 添加如下特性:

* 提升与 Hibernate 5.2 的兼容性.

* 通过[Query by Example](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#query-by-example)支持任意匹配.

* 分页查询优化.

* 支持 `exists`方法的识别.

### 2.2. Spring Data JPA 1.10 新特性

Spring Data JPA 1.10 添加了如下特性:

* 支持库查询方法的识别[Projections](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#projections).

* 支持[Query by Example](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#query-by-example)查询.

* 以下注解 `@EntityGraph`,`@Lock`,`@Modifying`,`@Query`,`@QueryHints`, 和`@Procedure`可用于复合注解.

* 集合表达式中支持`Contains`关键字.

* `AttributeConverter`implementations for`ZoneId`of JSR-310 and ThreeTenBP.

* 升级到 to Querydsl 4, Hibernate 5, OpenJPA 2.4, 和 EclipseLink 2.6.1.

## 3. 依赖

Due to the different inception dates of individual Spring Data modules, most of them carry different major and minor version numbers. The easiest way to find compatible ones is to rely on the Spring Data Release Train BOM that we ship with the compatible versions defined. In a Maven project, you would declare this dependency in the`<dependencyManagement />`section of your POM, as follows:

示例 1. 使用Spring Data release train BOM

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-releasetrain</artifactId>
      <version>Lovelace-SR10</version>
      <scope>import</scope>
      <type>pom</type>
    </dependency>
  </dependencies>
</dependencyManagement>
```

当前release train 版本是`Lovelace-SR10`. 当前可以使用的版本列表[点此了解](https://github.com/spring-projects/spring-data-commons/wiki/Release-planning). 版本名称格式为:`${name}-${release}`, ```${release}```的取值可能如下:

* `BUILD-SNAPSHOT`: 当前快照

* `M1`,`M2`, 等等: 里程碑

* `RC1`,`RC2`, 等等: 发布候选版

* `RELEASE`: 发布版

* `SR1`,`SR2`, 等等: 服务版本,升级版本

使用这种方式的示例[Spring Data examples repository](https://github.com/spring-projects/spring-data-examples/tree/master/bom). 有了这个你可以在```dependency```块定义spring data的模块而不使用版本号:

Example 2. 定义数据模块依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
  </dependency>
<dependencies>
```

### 3.1. Spring boot中定义依赖

Spring Boot 选择Spring Data模块的最近版本,如果你想升级到最新,只需配置`spring-data-releasetrain.version` 到你想用的版本即可.[可用的名字如下](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#dependencies.train-names)
S
### 3.2. Spring 框架

当前版本需要spring 版本 5.1.9.RELEASE 或者更高.这个模块在稍微旧的版本也能使用,但是强烈建议用较新的版本.

