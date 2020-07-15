---
title: maven的父子工程
date: 2020-07-15 16:54:55
categories: Maven
summary: maven父子工程
---

1、为什么需要父子工程项目？
* 当一个项目有多个模块，例如微服务。且这些模块有很多共同的依赖，比如mysql、druid、springboot等
* 这时可以建一个父工程，在父工程中管理子工程的共同依赖的版本，在各子工程中继承该父工程

2、简单的父子工程
* 父工程侧：
	* 创建一个空的工程，只有pom文件，来管理子工从的pom中依赖的版本
		* 注意：父工程的pom文件中的packaging标签应该设置为pom，表示打包方式为pom
		* 创建完必须进行install，将父工程发布到仓库中
	* 示例：

``` xml
<groupId>com.scu.parent</groupId>
<artifactId>parent</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>pom</packaging>   <!--这里声明打包的形式是pom-->


<!--这是声明子module，idea会自动填上-->
<modules>
  <module>childer-module</module>
</modules>

<!--统一管理jar包版本-->
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.source>1.8</maven.compiler.source>
  <maven.compiler.target>1.8</maven.compiler.target>
  <mysql.version>5.1.47</mysql.version>
</properties>

<!--子模块继承之后，提供作用：锁定版本+子module不用groupId和version-->
<dependencyManagement>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>${mysql.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```
- 子工程侧：

``` xml
<parent>
    <artifactId>parent</artifactId>
    <groupId>com.scu.parent</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
<modelVersion>4.0.0</modelVersion>

 
<artifactId>childer-module</artifactId>    <!--只声明artifactId，其他继承父pom-->


<dependencies>
    <!--mysql-connector-java-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>  <!--可以不指定版本-->
    </dependency>
</dependencies>
```

总结：子pom中如果没有声明版本，那么就去父pom上找对应版本


3、idea中创建父子工程：
* 1、创建一个空工程，写上pom文件。作为父工程
* 2、右击该root工程，创建module，表示在该父工程上创建子工程。idea会自动在父工程的pom文件中添加子module标签


4、解决maven单继承问题
* 在使用springboot中，官网说明要继承它们的依赖：
``` xml
<parent> 
    <groupId>org.springframework.boot</groupId> 
    <artifactId>spring-boot-starter-parent</artifactId> 
    <version>2.3.1.RELEASE</version> 
</parent>
```
* 但是这些子module已经继承了父工程了，由于maven的单继承，所以是不是不能再使用springboot了？
* 官网也给出了另一个引入依赖的办法（在父pom中）： [对应链接](https://docs.spring.io/spring-boot/docs/2.3.1.RELEASE/maven-plugin/reference/html/#using-import)
``` xml
<dependencyManagement>
  <dependencies>
    <!--spring boot 2.2.2-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>2.2.2.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```
* <type>pom</type>：表示不在父工程中引入依赖的jar包，只是加载这些依赖的信息，供子工程使用
