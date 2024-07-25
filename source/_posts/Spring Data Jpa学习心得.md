---

title: Spring Data Jpa 学习笔记

tag: spring

categories: 开发框架

date: 2024/5/2 20:46:25

index_img:  https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGVjMTI5ODM4YTlmOTgzMTFiOTQwOTMyOTY4MzM1NGZfVFU0a09CblhsQU04ODdtRDRvTTZjbHkySVZWU3QyNnNfVG9rZW46R2o4Q2I5Qm9Gb1pvNDZ4NUpjU2M1bWFsbndnXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA

---

> 官网 ；https://docs.spring.io/spring-data/jpa/reference/index.html

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGVjMTI5ODM4YTlmOTgzMTFiOTQwOTMyOTY4MzM1NGZfVFU0a09CblhsQU04ODdtRDRvTTZjbHkySVZWU3QyNnNfVG9rZW46R2o4Q2I5Qm9Gb1pvNDZ4NUpjU2M1bWFsbndnXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

# 一、spring Data Jpa 介绍

> **Jpa** : JPA（Java Persistence API）是用于在Java应用程序中进行对象持久化的API，用于描述对象-关系映射（ORM）规范（就像java的JDBC规范，每个数据库厂商都有对应的实现）。
>
> **Hibernate** ：Hibernate 是一个流行的开源对象-关系映射（ORM）框架，它实现了JPA（Java Persistence API）规范，为Java应用程序提供了数据持久化的解决方案。
>
> **Spring Data JPA**：Spring Data JPA 是 Spring 框架中的一个模块，它简化了对于基于 JPA (Java Persistence API) 的数据访问层的开发。JPA 是 Java EE 中用于对象持久化的标准，而 Spring Data JPA 则在其基础上提供了一系列便捷的功能，帮助开发者更轻松地与数据库进行交互。
>
> **Spring Data:** Spring Data是Spring生态中的一个项目，目标就是简化对数据访问的开发，个人觉得spring的野心很大，它并不限于关系型数据库，还支持文档型数据库，NoSQL数据库。

简单来说 Spring Data Jpa 是spring对Jpa规范的在一次封装，目的就是简化数据库操作。最大特点就是

**Defining Query Methods**定义查询方法。第一次接触的时候感觉很神奇。通过对方法名的解析就能自动生成查询语句。

# 二、快速入门

1. 引入maven坐标

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

1. 创建实体类 和 repository接口 还有启动类的开启Jpa ,代码中对应解释

```Java
package chances.learn.jpa.entity;

@Entity //@Entity是用来标识实体类，spring会把有这个注解的类
@Data
@Table(name="sc_person")  //对应数据库表名，也可以根据name生成属性
public class Person {

    @Id   //标识主键
    @GeneratedValue(strategy = GenerationType.IDENTITY)  //主键生成策略
    private Long id;
    private String name;
    private Integer age;
}

-- ------------------------------------------------------------------------
//Repository接口是 Jpa中最核心的接口，
// 继承之后例如 findByLastName(String lastName) 将会自动生成根据 lastName 查询实体的方法
package chances.learn.jpa.repo;
public interface PersonRepository extends Repository<Person, Long> {

  Person save(Person person);
  Optional<Person> findById(long id);
}

-- ------------------------------------------------------------------------
@SpringBootApplication
@EnableJpaRepositories(basePackages = "chances.learn.jpa.repo")
@EntityScan("chances.learn.jpa.repo.entity")
public class JpaLearnApplication {
    public static void main(String[] args) {
        SpringApplication.run(JpaLearnApplication.class, args);
    }
}
```

1. 我们测试一下看看结果 

```Java
@SpringBootTest
@Slf4j
class JpaLearnApplicationTests {
    @Resource
    PersonRepository personRepository;
    @Test
    void save() {
        Person person = new Person();
        person.setName("shichong_update");
        person.setAge(20);
        Person save = personRepository.save(person);
        log.info("sava result：{}",save.toString());
    }

}
```

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=MWQxMWVjMTFkZTcxYTJlMGMyZTkzNWZhZWIzNWE2ZmNfUDh1S0M4VWNZSURyc3dRYlVqd1V6QUx3NmhjSXJJN3pfVG9rZW46R2tFNWJQckhab0gxR1h4MlpiRmNtZGxwblpiXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDI2NTAzMDM3ZTRlMmZkZjY5YThhZTBiODk4YjE2ZDNfanpueGc0OTBhcWt0YjJpZXBTUnBMdm5hT1V2WWd5UkZfVG9rZW46V3ZTS2JEMVlEb2NJTnd4TW1VUWMyUFZXblllXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

# 三、核心接口

Spring Data抽象出来的顶层接口是` Repository`。 直接或间接实现这个接口才有操作数据的能力，不限于关系型数据库，还支持Redis ，MangoDB等。

Spring data jpa 通过解析方法名自动去生成查询sql。为了简化开发者开发，也提供了一些带有常用查询方法的接口，我们只需要直接实现就可以使用。例如CrudRepository ，ListCrudRepository ,JpaRepository,PagingAndSortingRepository等接口，每一个接口都补充了一些特定的功能下面是我在idea中查看的关系图， 

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDU1ZmY3MTZlNjYxZDI3ODkyMGQyZGZlZDYwZGE5ODhfdkNPOElZOWN5MFphajhZMnNyUkFNZzA4V3BxU3MwSUFfVG9rZW46WGZPSWI0dFMwb3EyYjd4eWdNNWNLOGp3blRmXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

## 3.1 CrudRepository接口 增删改查接口

 对比实现普通Repository和CrudRepository，看一看UserRepository会发生什么变化

- 实现Repository。

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=MWFjODNhNTlhM2YwODU3OGRjNjVhNTUxZWIyYjFiNzhfYTlKNHN3S0xJT3lIMDF5eFdQVE5Qd2szS01nWllwT3BfVG9rZW46UTVVTWJxRUJEb2kyUm94UE5zR2NjOFZSbk5kXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

- 实现CrudRepository

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=NTBiODk3M2M5NjRiNjM3ZGFiOTdiZTI0MjBhOThkMGJfNGFmNFdPaUdaZVhHMTBLR0d4TlppNlQyZHBDM3ZBQW5fVG9rZW46SUpKMmJERjJob2pkZjd4R3lEWWN0OEZ2bklnXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

- 总结，CrudRepository就和我们自己定义的repository是一样的原理，只不过声明好了一些通用的方法，搭配泛型就可以复用一些通用的操作.看一下源码

```Java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package org.springframework.data.repository;

import java.util.Optional;

@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);

    <S extends T> Iterable<S> saveAll(Iterable<S> entities);

    Optional<T> findById(ID id);

    boolean existsById(ID id);

    Iterable<T> findAll();

    Iterable<T> findAllById(Iterable<ID> ids);

    long count();

    void deleteById(ID id);

    void delete(T entity);

    void deleteAllById(Iterable<? extends ID> ids);

    void deleteAll(Iterable<? extends T> entities);

    void deleteAll();
}
```

- @NoRepositoryBean：  因为继承了repository接口spring会认为他也是一个bean，实际并不是只是中间扩展接口，所以使用@NoRepositoryBean标识一下 就不会当作Bean处理了

支持分页接口

## 3.2 PagingAndSortingRepository 分页排序接口

1. 使用的时候直接继承这个接口即可,声明一个方法返回结果为 page 字面意思理解就是
   1. ```Java
      package chances.learn.jpa.repo;
      
      import chances.learn.jpa.entity.Person;
      import org.springframework.data.domain.Page;
      import org.springframework.data.domain.Pageable;
      import org.springframework.data.repository.CrudRepository;
      import org.springframework.data.repository.PagingAndSortingRepository;
      import org.springframework.data.repository.Repository;
      
      import java.util.Optional;
      
      public interface PersonRepository extends CrudRepository<Person, Long>, PagingAndSortingRepository<Person,Long> {
      
        Person save(Person person);
        
        Page<Person> findAll(String name, Pageable pageable);
      
        Optional<Person> findById(long id);
      
      }
      ```
2. 开始unit test ，创建分页查询条件时候 使用PageRequest.of方法 。看看数据库和结果。
   1. ![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=MTliOTRmNmQyMDA0ZDZlNGQ2NzFiNjMyYjQ2OWU5YjFfenZtMFJzTlg0R3hKNmlWTno4SWcxTWJBZ2xlUkQ3ZkNfVG9rZW46RDdlRmJsMUFOb0tuVlJ4eWxUS2N2YWRhbjJkXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

   2. ![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTI4YmM1NzA2ZGExMzMzYzE0ZDZhYTUyYTliYmM2YjBfUGhyOUMzUERzQU1XMFZKZk9vWVljM0Ezb3lpSWNiZFRfVG9rZW46TFVCamJmbWpZb3BnaTZ4MXFPcWNFa3hzbmNoXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

   3. ```Java
      @Test
      void page() {
          //每页两条 第一页 按照age降序
          Sort sort = Sort.by(Sort.Direction.DESC, "age");
          Page<Person> all = personRepository.findAll(PageRequest.of(0, 2));
          log.info("page result：{}",all.toString());
      }
      ```

## 3.3 JpaRepository 整合了常用的数据库操作

通过扩展 `JpaRepository`，继承基本的 CRUD（创建、读取、更新、删除）操作，以及使用 Spring Data 的本地查询进行分页、排序方法。这个接口简化了 Spring 应用程序中数据访问层的开发，减少了与数据库操作通常相关的样板代码。

在debug的时候发现一件事情，发现PersonRepository经过动态代理之后已经变成了SimpleJpaRepository类型，

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=MTRjZWU3ODZjMDIxNDEwNjJhNGU1ZjhjZjEzZGQxNzVfamFsRDNRU2VBNGtWMXFOZUY2V2ZOT0sySnhCVFU5N0hfVG9rZW46TlJyemJhUTFab0ZDbWt4YUdPRGM0NHZvbmhnXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

# 四、Defining Query Methods 定义查询方法（Jpa特色）

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGI3MDM5ZDY3MzJhYzI1OGJiZGYzNjc3NDNjMGVjZjNfSXlEdG5YQmxGR3d6dmxRWHZ4VnQySjNLMG5LaFo2VXVfVG9rZW46VXlXT2JGS0ZSb3J1ZTl4UXRZMGNhQWt5bmhjXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

Defining Query Method是Jpa的一大特性。定义查询方法有官网说明**提供了两种方式**。第一种 通过直接从方法名称解析后查询，第二种 通过手动定义方式的来执行。

简单示例

```Java
//第一种 没有sql直接解析方法名
Optional<LoginToken> findByUserId(String userId);

//第二种
@Query("select e from SysUser e where e.userName = :userName and e.deleted = false")
Optional<SysUser> findByUserName(@Param("userName") String userName);
```

## 4.1 Query Lookup Strategies查询查找策略

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGU4ZDI1MGFkNTRlNTU3ZjRmNmY1MDdmMGIwNmEyMTRfeld5UWpiQ0lPb2ZWOWlHVElLUVE2VUN3Z3V0NmtEQ1VfVG9rZW46Unk5S2JJTFE3b0lWZG14dDVFRGNPaWJwblFmXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)

在 Spring Data JPA 中，有两种主要的方式来定义查询方法：基于方法名的命名约定和使用 `@Query` 注解定义的自定义查询。查询查找策略决定了 Spring Data JPA 在执行方法时应该优先考虑哪种方式。官网中告诉我们一共有三种选择策略模式，和如何切换查找策略。

- `CREATE` 从方法名解析删除指定的前缀 例如 find,findAll，然后解析方法名的其余部分。也就是直接根据方法名创建查询。如果方法名不符合规则启动会报异常，及时配置了 `@query` 注解
- *`USE_DECLARED_QUERY`* *使用声明好的查询，也就是使用**`@query`* *注解声明的查询，如果没找到也会抛出异常。*
- `CREATE_IF_NOT_FOUND` 这是默认的查找策略，先去使用`@Quer`y声明好的查询，如果找不到再去使用通过方法名创建查询。

更改默认的策略修改`@EnableJpaRepositories`的queryLookupStrategy属性即可。

```Java
package chances.learn.jpa;

@SpringBootApplication
@EnableJpaRepositories(basePackages="chances.learn.jpa.repo", queryLookupStrategy= QueryLookupStrategy.Key.CREATE)
@EntityScan("chances.learn.jpa.entity")
public class JpaLearnApplication {
    public static void main(String[] args) {
        SpringApplication.run(JpaLearnApplication.class, args);
    }
}
```

## 4.2 通过方法名解析查询

下面是官网给出的示例，可以看出方法名是通过**前缀+字段名+字段描述符**+...组成

```Java
interface PersonRepository extends Repository<Person, Long> {
    
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

前缀不仅有find还有 count，delete,remove等其他的前缀。每个版本可能支持的都不一样可以看看Jpa的查询解析语法树看看。 `org.springframework.data.repository.query.parser.PartTree`

```
org.springframework.data.repository.query.parser.Part.Type
```

![img](https://chances.feishu.cn/space/api/box/stream/download/asynccode/?code=YWJhNDAzMmM3ZTkyZjBjNzA5ZDBkNjNiZTRhNzJhZDRfeldSa1QzU0xySkNyVFd6cWtsc01kNE9GclpHOWFkWFhfVG9rZW46TlVWNGJsS09Eb0NlQ1p4T2VNb2NZY3FwbklnXzE3MTQ5MDIzMzg6MTcxNDkwNTkzOF9WNA)