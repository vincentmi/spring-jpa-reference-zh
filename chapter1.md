# 前言

Spring Data JPA  提供了JAVA持久化API \(即JPA\)的库,用于简化需要访问JPA数据源的应用程序的开发

## 1. 项目资料 {#project}

* 版本库 -[https://github.com/spring-projects/spring-data-jpa](https://github.com/spring-projects/spring-data-jpa)

* BUG跟踪 -[https://jira.spring.io/browse/DATAJPA](https://jira.spring.io/browse/DATAJPA)

* 发布包 -[https://repo.spring.io/libs-release](https://repo.spring.io/libs-release)

* 里程碑包 -[https://repo.spring.io/libs-milestone](https://repo.spring.io/libs-milestone)

 repository -[https://repo.spring.io/libs-snapshot](https://repo.spring.io/libs-snapshot)

## 2. New & Noteworthy {#new-features}

### 2.1. What’s New in Spring Data JPA 1.11 {#new-features.1-11-0}

Spring Data JPA 1.11 added the following features:

* Improved compatibility with Hibernate 5.2.

* Support any-match mode for[Query by Example](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#query-by-example).

* Paged query execution optimizations.

* Support for the`exists`projection in repository query derivation.

### 2.2. What’s New in Spring Data JPA 1.10 {#new-features.1-10-0}

Spring Data JPA 1.10 added the following features:

* Support for[Projections](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#projections)in repository query methods.

* Support for[Query by Example](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#query-by-example).

* The following annotations have been enabled to build on composed annotations:`@EntityGraph`,`@Lock`,`@Modifying`,`@Query`,`@QueryHints`, and`@Procedure`.

* Support for the`Contains`keyword on collection expressions.

* `AttributeConverter`implementations for`ZoneId`of JSR-310 and ThreeTenBP.

* Upgrade to Querydsl 4, Hibernate 5, OpenJPA 2.4, and EclipseLink 2.6.1.

## 3. Dependencies {#dependencies}

Due to the different inception dates of individual Spring Data modules, most of them carry different major and minor version numbers. The easiest way to find compatible ones is to rely on the Spring Data Release Train BOM that we ship with the compatible versions defined. In a Maven project, you would declare this dependency in the`<dependencyManagement />`section of your POM, as follows:

Example 1. Using the Spring Data release train BOM

```
<
dependencyManagement
>
<
dependencies
>
<
dependency
>
<
groupId
>
org.springframework.data
<
/groupId
>
<
artifactId
>
spring-data-releasetrain
<
/artifactId
>
<
version
>
Lovelace-SR10
<
/version
>
<
scope
>
import
<
/scope
>
<
type
>
pom
<
/type
>
<
/dependency
>
<
/dependencies
>
<
/dependencyManagement
>
```

The current release train version is`Lovelace-SR10`. The train names ascend alphabetically and the currently available trains are listed[here](https://github.com/spring-projects/spring-data-commons/wiki/Release-planning). The version name follows the following pattern:`${name}-${release}`, where release can be one of the following:

* `BUILD-SNAPSHOT`: Current snapshots

* `M1`,`M2`, and so on: Milestones

* `RC1`,`RC2`, and so on: Release candidates

* `RELEASE`: GA release

* `SR1`,`SR2`, and so on: Service releases

A working example of using the BOMs can be found in our[Spring Data examples repository](https://github.com/spring-projects/spring-data-examples/tree/master/bom). With that in place, you can declare the Spring Data modules you would like to use without a version in the`<dependencies />`block, as follows:

Example 2. Declaring a dependency to a Spring Data module

```
<
dependencies
>
<
dependency
>
<
groupId
>
org.springframework.data
<
/groupId
>
<
artifactId
>
spring-data-jpa
<
/artifactId
>
<
/dependency
>
<
dependencies
>
```

### 3.1. Dependency Management with Spring Boot {#dependencies.spring-boot}

Spring Boot selects a recent version of Spring Data modules for you. If you still want to upgrade to a newer version, configure the property`spring-data-releasetrain.version`to the[train name and iteration](https://docs.spring.io/spring-data/jpa/docs/2.1.10.RELEASE/reference/html/#dependencies.train-names)you would like to use.

### 3.2. Spring Framework {#dependencies.spring-framework}

The current version of Spring Data modules require Spring Framework in version 5.1.9.RELEASE or better. The modules might also work with an older bugfix version of that minor version. However, using the most recent version within that generation is highly recommended.

