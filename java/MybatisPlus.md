> 由于单表的CRUD功能代码重复度很高，也没有什么难度。而这部分代码量往往比较大，开发起来比较费时。可以使用MybatisPlus来简化开发，提升开发效率

### 引入依赖、编写配置文件

mybatisplus的依赖坐标如下：

```XML
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.3.1</version>
</dependency>
```

再启动类上配置mybatisplus的扫描范围：

```Java
@MapperScan("com.hjl.project.mapper")
@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

在application.yml配置文件里的配置：配置type-aliases-package​属性是为了在编写自定义的XML映射文件（例如 UserMapper.xml​）时，可以简化代码。比如配置了别名后，在XML中就不需要写实体类的全限定名（如 com.example.entity.User​），而可以直接使用类名（如 User​）

```yml
mybatis-plus:
  # 配置扫描 mapper.xml 文件的位置，默认也是这个位置，可以省略
  mapper-locations: classpath*:/mapper/**/*.xml
  # 配置实体类所在的包
  type-aliases-package: com.example.mybatisplus.model.entity
  global-config:
    db-config:
      id-type: assign_id # 全局主键策略。ASSIGN_ID（雪花算法）适用于分布式，AUTO（数据库自增）适用于单机。
      table-prefix: t_ # 表名前缀。如果所有表都有共同前缀（如`t_user`），配置后实体类名可简写为`User`。
      logic-delete-field: is_deleted # 全局逻辑删除字段名（需要数据库有该字段）
      logic-delete-value: 1 # 逻辑已删除值
      logic-not-delete-value: 0 # 逻辑未删除值
  configuration:
    # 开启驼峰命名转换
    map-underscore-to-camel-case: true
    # 日志实现
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

---

### 实体类上的注解使用

> MybatisPlus会根据PO实体的信息来推断出表的信息，从而生成SQL的。在使用默认实现与实际场景不同时，可以使用注解来处理，下面是一些mybatisplus的默认实现：
> 
> - MybatisPlus会把PO实体的类名驼峰转下划线作为表名
> - MybatisPlus会把PO实体的所有变量名驼峰转下划线作为表的字段名，并根据变量类型推断字段类型
> - MybatisPlus会把名为id的字段作为主键

#### @TableName

> 表名注解，标识实体类对应的表

示例代码：

```Java
@TableName("user")
public class User {
    private Long id;
    private String name;
}
```

属性值：

|属性|类型|必须指定|默认值|描述|
|---|---|---|---|---|
|value|String|否|""|表名|
|schema|String|否|""|schema|
|keepGlobalPrefix|boolean|否|false|是否保持使用全局的 tablePrefix 的值（当全局 tablePrefix 生效时）|
|resultMap|String|否|""|xml 中 resultMap 的 id（用于满足特定类型的实体类对象绑定）|
|autoResultMap|boolean|否|false|是否自动构建 resultMap 并使用（如果设置 resultMap 则不会进行 resultMap 的自动构建与注入）|
|excludeProperty|String[]|否|{}|需要排除的属性名 @since 3.3.1|

#### @TableId

> 主键注解，标识实体类中的主键字段

示例代码：

```Java
@TableName("user")
public class User {
    @TableId
    private Long id;
    private String name;
}
```

属性值：

|属性|类型|必须指定|默认值|描述|
|---|---|---|---|---|
|value|String|否|""|表名|
|type|Enum|否|IdType.NONE|指定主键类型|

IdType属性的值有多个

|值|描述|
|---|---|
|AUTO|数据库 ID 自增|
|NONE|无状态，该类型为未设置主键类型（注解里等于跟随全局，全局里约等于 INPUT）|
|INPUT|insert 前自行 set 主键值|
|ASSIGN_ID|分配 ID(主键类型为 Number(Long 和 Integer)或 String)(since 3.3.0),使用接口IdentifierGenerator的方法nextId(默认实现类为DefaultIdentifierGenerator雪花算法)|
|ASSIGN_UUID|分配 UUID,主键类型为 String(since 3.3.0),使用接口IdentifierGenerator的方法nextUUID(默认 default 方法)|

#### @TableField

> 普通字段注解

使用该注解的场景：

- 成员变量名与数据库字段名不一致
- 成员变量是以isXXX​命名，按照JavaBean​的规范，MybatisPlus​识别字段时会把is​去除，这就导致与数据库不符。
- 成员变量名与数据库一致，但是与数据库的关键字冲突。使用@TableField​注解给字段名添加转义字符：``​

示例代码：

```Java
@TableName("user")
public class User {
    @TableId
    private Long id;
    private String name;
    private Integer age;
    @TableField("is_married")
    private Boolean isMarried;
    @TableField("`concat`")
    private String concat;
}
```

属性值：只列举常用的

|属性|类型|必填|默认值|描述|
|---|---|---|---|---|
|value|String|否|""|数据库字段名|
|exist|boolean|否|true|是否为数据库表字段|

---

### 在Mapper层的使用

继承`BaseMapper<T>`​类，即可使用对应数据库表的常见CRUD方法了

```java
package com.itheima.mp.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.hjl.project.model.entity.User;

public interface UserMapper extends BaseMapper<User> {
}
```

对应的方法，增、删、改、查对应的命名是insert​、delete​、update​、select​

​![image](assets/image-20251226204552-1qe7sag.png)​

---

### Wrapper

> 若要使用更加复杂的查询需要使用构造器Wrapper​

​![image](assets/image-20251226211734-giva9pq.png)​

​AbstractWrapper​提供了where​中所有的条件构造方法：  
​![image](assets/image-20251226213254-zhv5gqk.png)​

#### QueryWrapper

> 无论是删、改、查操作，都可以使用QueryWrapper来构建查询条件

QueryWrapper在AbstractWrapper的基础上拓展了一个select​方法，允许指定查询字段：  
​![image](assets/image-20251226213402-a5kt8ua.png)  
查询操作示例代码如下：

```Java
@Test
void testQueryWrapper() {
    // 1.构建查询条件 where name like "%o%" AND balance >= 1000
    QueryWrapper<User> wrapper = new QueryWrapper<User>()
            .select("id", "username", "info", "balance")
            .like("username", "o")
            .ge("balance", 1000);
    // 2.查询数据
    List<User> users = userMapper.selectList(wrapper);
    users.forEach(System.out::println);
}
```

更新操作示例代码如下：

> BaseMapper里提供的update()方法，第一个参数表示要更新的值，也就是sql语句中的"set"部分。规则就是传入的实体类对象中的非null字段会作为set语句，这里就有个注意点：因为int、float、long这样的基本数据类型的默认值通常不为null，所以会被mybatisplus放入到set语句中更新，这就出问题了，所以实体类的字段定义要为包装类类型。

```Java
@Test
void testUpdateByQueryWrapper() {
    // 1.构建查询条件 where name = "Jack"
    QueryWrapper<User> wrapper = new QueryWrapper<User>().eq("username", "Jack");
    // 2.更新数据，user中非null字段都会作为set语句
    User user = new User();
    user.setBalance(2000);
    userMapper.update(user, wrapper);
}
```

#### UpdateWrapper

> 由于基于BaseMapper中的update方法更新时只能直接赋值，对于一些复杂的需求就难以实现。若SET的赋值结果是基于字段现有值的，这个时候就要利用UpdateWrapper中的setSql功能了

UpdateWrapper在AbstractWrapper的基础上拓展了一个set​方法，允许指定SQL中的SET部分：  
​![image](assets/image-20251226213451-bdm6nc9.png)  
示例代码如下：

```Java
//预期sql：UPDATE user SET balance = balance - 200 WHERE id in (1, 2, 4)
@Test
void testUpdateWrapper() {
    List<Long> ids = List.of(1L, 2L, 4L);
    // 1.生成SQL
    UpdateWrapper<User> wrapper = new UpdateWrapper<User>()
            .setSql("balance = balance - 200") // SET balance = balance - 200
            .in("id", ids); // WHERE id in (1, 2, 4)
        // 2.更新，注意第一个参数可以给null，也就是不填更新字段和数据，
    // 而是基于UpdateWrapper中的setSQL来更新
    userMapper.update(null, wrapper);
}
```

#### LambdaQueryWrapper、LambdaUpdateWrapper

> 在使用QueryWrapper、UpdateWrapper构建查询条件时，要直接写死字段名称，这在编程规范中不规范，所以可以使用LamdaQueryWrapper、LambdaUpdateWrapper来改善

可以基于变量的gettter​方法结合反射技术。因此我们只要将条件对应的字段的getter​方法传递给MybatisPlus，它就能计算出对应的变量名了。而传递方法可以使用JDK8中的方法引用​和Lambda​表达式

```Java
@Test
void testLambdaQueryWrapper() {
    // 1.构建条件 WHERE username LIKE "%o%" AND balance >= 1000
          LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
            .select(User::getId, User::getUsername, User::getInfo, User::getBalance)
            .like(User::getUsername, "o")
            .ge(User::getBalance, 1000);
    // 2.查询
    List<User> users = userMapper.selectList(wrapper);
    users.forEach(System.out::println);
}
```

#### 在自定义sql里使用mybatisplus生成的语句的方法

> 在有些场景下，需要自己编写sql语句，但是部分sql语句很繁琐又可以用mybatisplus生成，那么这时候还是可以使用${ew.customSqlSegment}来拼接

准备好wrapper，并作为参数传递给mapper层

```Java
@Test
void testCustomWrapper() {
    // 1.准备自定义查询条件
    List<Long> ids = List.of(1L, 2L, 4L);
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>().in(User::getId, ids);

    // 2.调用mapper的自定义方法，直接传递Wrapper
    userMapper.deductBalanceByIds(200, wrapper);
}
```

然后再自定义sql里拼接语句

```Java
public interface UserMapper extends BaseMapper<User> {
    @Select("UPDATE user SET balance = balance - #{money} ${ew.customSqlSegment}")
    void deductBalanceByIds(@Param("money") int money, @Param("ew") LambdaQueryWrapper<User> wrapper);
}
```

---

### 在Service层的使用

> MybatisPlus不仅提供了BaseMapper，还提供了通用的Service接口及默认实现，封装了一些常用的service模板方法。 通用接口为IService​，默认实现为ServiceImpl​，其中封装的方法可以分为以下几类：
> 
> - ​save​：新增
> - ​remove​：删除
> - ​update​：更新
> - ​get​：查询单个结果
> - ​list​：查询集合结果
> - ​count​：计数
> - ​page​：分页查询

#### Service层中的CRUD方法

##### 新增

​![image](assets/image-20251227135137-t7mgs73.png)​

- ​save​是新增单个元素
- ​saveBatch​是批量新增
- ​saveOrUpdate​是根据id判断，如果数据存在就更新，不存在则新增
- ​saveOrUpdateBatch​是批量的新增或修改

##### 查询

- get
    
    ​![image](assets/image-20251227135743-or1bu9i.png)​
    
    - ​getById​：根据id查询1条数据
    - ​`getOne(Wrapper<T>)`​：根据Wrapper​查询1条数据
    - ​getBaseMapper​：获取Service​内的BaseMapper​实现，某些时候需要直接调用Mapper​内的自定义SQL​时可以用这个方法获取到Mapper​

- list
    
    ​![image](assets/image-20251227135840-qsr5vkp.png)​
    
    - ​listByIds​：根据id批量查询
    - ​`list(Wrapper<T>)`​：根据Wrapper条件查询多条数据
    - ​list()​：查询所有
- count
    
    ​![image](assets/image-20251227140040-2caevye.png)​
    
    - ​count()​：统计所有数量
    - ​`count(Wrapper<T>)`​：统计符合Wrapper​条件的数据数量

##### 更新

​![image](assets/image-20251227135426-4kbvcyb.png)​

- ​updateById​：根据id修改
- `​update(Wrapper<T>)`​：根据UpdateWrapper​修改，Wrapper​中包含set​和where​部分
- ​`update(T，Wrapper<T>)`​：按照T​内的数据修改与Wrapper​匹配到的数据
- ​updateBatchById​：根据id批量修改

##### 删除

​![image](assets/image-20251227135333-5lm6eek.png)​

- ​removeById​：根据id删除
- ​removeByIds​：根据id批量删除
- ​removeByMap​：根据Map中的键值对为条件删除
- ​`remove(Wrapper<T>)`​：根据Wrapper条件删除

#### lambdaQuery、lambdaUpdate

> 在实现复杂查询条件的时候，IService提供了对比Wrapper更加简化的方法，不需要常见Wrapper对象，直接使用lambdaQuery、lambdaUpdate即可实现

- 使用LambdaQueryWrapper
    
    ```Java
    @GetMapping("/list")
    public List<UserVO> queryUsers(UserQuery query){
        // 1.组织条件
        String username = query.getName();
        Integer status = query.getStatus();
        Integer minBalance = query.getMinBalance();
        Integer maxBalance = query.getMaxBalance();
        LambdaQueryWrapper<User> wrapper = new QueryWrapper<User>().lambda()
                .like(username != null, User::getUsername, username)
                .eq(status != null, User::getStatus, status)
                .ge(minBalance != null, User::getBalance, minBalance)
                .le(maxBalance != null, User::getBalance, maxBalance);
        // 2.查询用户
        List<User> users = userService.list(wrapper);
        // 3.处理vo
        return BeanUtil.copyToList(users, UserVO.class);
    }
    ```
    
- 使用lambQuery
    
    ```Java
    @GetMapping("/list")
    public List<UserVO> queryUsers(UserQuery query){
        // 1.组织条件
        String username = query.getName();
        Integer status = query.getStatus();
        Integer minBalance = query.getMinBalance();
        Integer maxBalance = query.getMaxBalance();
        // 2.查询用户
        List<User> users = userService.lambdaQuery()
                .like(username != null, User::getUsername, username)
                .eq(status != null, User::getStatus, status)
                .ge(minBalance != null, User::getBalance, minBalance)
                .le(maxBalance != null, User::getBalance, maxBalance)
                .list();
        // 3.处理vo
        return BeanUtil.copyToList(users, UserVO.class);
    }
    
    ```
    
    在lambdaQuery方法中除了可以构建条件，还需要在链式编程的最后添加一个list()​，这是在告诉MP调用结果需要是一个list集合。这里不仅可以用list()​，可选的方法还有：
    
    - ​.one()​：最多1个结果
    - ​.list()​：返回集合结果
    - ​.count()​：返回计数结果
- lambdaUpdate
    
    ```Java
    @Override
    @Transactional
    public void deductBalance(Long id, Integer money) {
        // 1.查询用户
        User user = getById(id);
        // 2.校验用户状态
        if (user == null || user.getStatus() == 2) {
            throw new RuntimeException("用户状态异常！");
        }
        // 3.校验余额是否充足
        if (user.getBalance() < money) {
            throw new RuntimeException("用户余额不足！");
        }
        // 4.扣减余额 update tb_user set balance = balance - ?
        int remainBalance = user.getBalance() - money;
        lambdaUpdate()
                .set(User::getBalance, remainBalance) // 更新余额
                .set(remainBalance == 0, User::getStatus, 2) // 动态判断，是否更新status
                .eq(User::getId, id)
                .eq(User::getBalance, user.getBalance()) // 乐观锁
                .update();
    }
    ```
    

---

### 何时使用BaseMapper？何时使用IService？

> 简单来说，可以把 BaseMapper​看作是提供了各种基础工具和特殊形状接头的工具箱，而 IService​则是在此基础上，提供了针对常见业务场景（如批量组装、一键操作）的电动工具。
> 
> - 处理​特定、非标准的数据查询​（如只取一个字段）或需要最基础的原子操作时，考虑使用 BaseMapper​的特有方法。
> - 处理​常规业务逻辑​，尤其是涉及批量处理、事务和便捷操作时，务必优先使用 IService​ 的方法。

#### 📊 特有方法对比

| 层级/接口              | 特有方法举例                                                     | 核心用途与场景                                                        |
| ------------------ | ---------------------------------------------------------- | -------------------------------------------------------------- |
| DAO层:BaseMapper​   | ​List<Object> selectObjs(Wrapper<T> wrapper)​              | 查询并只返回第一个字段的值的列表，适用于只需要获取单个字段（如ID、名称）的场景，结果非常轻量。               |
|                    | ​List<Map<String, Object>> selectMaps(Wrapper<T> wrapper)​ | 查询结果以List<Map>​形式返回，适用于无需转换成实体对象的统计查询、动态字段查询或结果集直接转为JSON输出的情况。 |
|                    | ​T selectOne(Wrapper<T> wrapper)​                          | ​严格查询单条记录​。如果结果多于一条，会抛出异常，适用于根据唯一键查询的场景。                       |
|                    | ​int deleteByMap(Map<String, Object> columnMap)​           | 根据简单的字段-值Map条件进行删除，适合等值条件删除。                                   |
| Service层:IService​ | ​boolean saveOrUpdate(T entity)​                           | ​新增或更新​。根据实体对象的主键是否存在自动判断操作类型，是业务中非常高频和便捷的方法。                  |
|                    | ​boolean saveBatch(Collection<T> entityList)​              | ​批量插入​。内部采用分批提交，性能远高于循环调用单次插入，是Service层最核心的优势之一。               |
|                    | ​boolean updateBatchById(Collection<T> entityList)​        | ​根据ID批量更新​。同样有分批处理机制，用于批量更新不同ID对象的不同字段。                        |
|                    | ​boolean removeByIds(Collection<?> idList)​                | ​根据ID集合批量删除​。相较于Mapper的deleteByIds​，返回布尔值更符合业务逻辑判断习惯。          |
||​T getOne(Wrapper<T> wrapper, boolean throwEx)​|增强版的查询单条。可控制当结果不唯一时​是否抛出异常​，比selectOne​更安全可控。|
||链式查询/更新​ (如.query().eq(...).list()​)|提供非常优雅的​链式API​，可以在一行代码内完成复杂条件构造，提升代码可读性。|

#### 💡 如何选择：场景与最佳实践

理解这些特有方法后，选择使用哪个就非常清晰了：

1. 何时使用 BaseMapper​的特有方法？
    
    - 执行特定类型的查询：当你需要查询结果不是标准实体对象列表时，应直接使用 BaseMapper​的方法。例如， selectObjs​用于获取单个字段值的列表，或者 selectMaps​用于直接获取可序列化的Map结果。
    - ​在自定义的Mapper方法中​：当你在Service实现类中编写复杂业务逻辑，需要组合多个基础操作时，可以通过 getBaseMapper()​调用这些更底层的方法。
    - ​追求极致的性能控制​：在极少数需要精细控制SQL执行细节的场景下，直接使用 BaseMapper​的原子操作。
2. 何时应优先使用 IService​的特有方法？
    
    - ​绝大多数业务逻辑层代码​：在Controller或自定义Service方法中，​应优先使用 IService​的方法​。它的方法设计（如返回boolean​）更符合业务语义，且批量操作方法能显著提升性能。
    - ​需要批量操作时​：这是最强烈的使用场景。saveBatch​, updateBatchById​等方法内置了分批处理逻辑，是 BaseMapper​所不具备的核心优势，能有效避免大数据量操作时的性能问题。
    - ​需要“保存或更新”逻辑时​：saveOrUpdate​方法极大地简化了这种常见业务逻辑。
    - 希望代码更简洁时：链式操作（如 service.lambdaQuery().eq(...).list()​）让条件构造和查询一气呵成，可读性非常高。

---

### 代码自动生成

> 因为实体类确认好后，mapper和service的构造是很固定的，所以为了提高开发效率，直接使用工具来生成框架，这里使用的工具是MybatisX

#### 1、编写库表

编写表单的时候注意每个字段的书写规范：考虑默认值、是否为空、注解、是否需要为某些字段添加索引

```sql
-- 题目表
create table if not exists question
(
    id          bigint auto_increment comment 'id' primary key,
    userId      bigint                             not null comment '创建用户 id',
    title       varchar(512)                       null comment '标题',
    content     text                               null comment '内容',
    tags        varchar(1024)                      null comment '标签列表（json 数组）',
    answer      text                               null comment '题目答案',
    submitNum   int      default 0                 not null comment '题目提交数',
    acceptedNum int      default 0                 not null comment '题目通过数',
    judgeConfig text                               null comment '判题配置（json 数组）',
    judgeCase   text                               null comment '判题用例（json 对象）',
    createTime  datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime  datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete    tinyint  default 0                 not null comment '是否删除',
    index idx_userId (userId)
) comment '题目' collate = utf8mb4_unicode_ci;
```

#### 2、使用MybatisX插件自动生成实体类、mapper接口、service接口和实体类

1）创建好库表后，右键对应的表使用MybatisX-Generator功能  
​![image](assets/image-20251227160347-61hmw2d.png)​

2）选择自己需要的功能  
​![image](assets/image-20251227161659-1qtjgkv.png)|  
​![image](assets/image-20251227163048-tdabjuo.png)​

#### 3、添加到自己的项目中

再将生成的代码检查一下，重构到自己的代码中