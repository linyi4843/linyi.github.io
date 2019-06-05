---
layout:     post
title:      "mybatis问题"
subtitle:   " \"mybatis\""
date:       2019-6-5 21:07:00
author:     "linyi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags:
    - mybatis
---

> “Never give up. ”



## mybatis缓存

## 一级缓存

   一级缓存是sqlSession级别的,每一次操作数据库的时候都会构造一个sqlSession对象,所以不同的操作sqlSession是不同的,也是互相不想影响的
当在同一个sqlSession内执行两次重复的sql的时候,第一次查询的数据会被写入缓存中(一个HashMap存在内存中),而第二次相同查询的时候会直接从缓存中取出缓存数据
提高查询效率<br> 
   但当sqlSession执行DML操作时候,就会清空缓存,下次查询就会从数据库查询再写入缓存中,避免出现脏读现象,且当sqlSession结束时,缓存也会消失  

## 二级缓存
   二级缓存默认是不开启的,需要手动配置(mapper.xml)<br>
        <cache/ > <br>
        默认属性如下 : 
        
        映射语句文件中的所有 select 语句将会被缓存。
        映射语句文件中的所有 insert,update 和 delete 语句会刷新缓存。
        缓存会使用 Least Recently Used(LRU,最近最少使用的)算法来收回。
        根据时间表(比如 no Flush Interval,没有刷新间隔), 缓存不会以任何时间顺序来刷新。
        缓存会存储列表集合或对象(无论查询方法返回什么)的 1024 个引用。
        缓存会被视为是 read/write(可读/可写)的缓存,意味着对象检索不是共享的,而且可以安全地被调用者修改,而不干扰其他调用者或线程所做的潜在修改。
   
   也可以自定义二级缓存<br>
     
      eviction：缓存回收策略
      - LRU：最少使用原则，移除最长时间不使用的对象
      - FIFO：先进先出原则，按照对象进入缓存顺序进行回收
      - SOFT：软引用，移除基于垃圾回收器状态和软引用规则的对象
      - WEAK：弱引用，更积极的移除移除基于垃圾回收器状态和弱引用规则的对象
      flushInterval：刷新时间间隔，单位为毫秒，这里配置的100毫秒。如果不配置，那么只有在进行数据库修改操作才会被动刷新缓存区
      size：引用额数目，代表缓存最多可以存储的对象个数
      readOnly：是否只读，如果为true，则所有相同的sql语句返回的是同一个对象（有助于提高性能，但并发操作同一条数据时，可能不安全），如果设置为false，则相同的sql，后面访问的是cache的clone副本。
      
   二级缓存的作用于是整个mapper,比sqlSession的作用域更大,多个sqlSession可以共享一个mapper,当多个sqlSession操作同一个mapper下的sql语句
且参数也相同,在第一个sqlSession查询之后,会将一级缓存升级为二级缓存,下次任意sqlSession执行当前mapper同样的sql且参数相同,会直接从二级缓存中拿数据<br>
    同样DML操作是,也会清空缓存 <br>
    <br>
    ps:( 多表操作不能使用二级缓存，这样会导致数据不一致的问题)

## 延迟加载
  将采用高级映射(resultMap)实现多表联查时向数据库发出的SQL语句拆分成若干条单表查询的SQL语句，当需要返回数据时才会向数据库发出只针对当前数据的SQL语句,提高查询效率。<br>
  也需要手动配置 
  ```
  <!--全局参数设置-->
  <settings>
      <!--延迟加载总开关-->
      <setting name="lazyLoadingEnabled" value="true"/>
      <!--侵入式延迟加载开关-->
      <!--3.4.1版本之前默认是true，之后默认是false-->
      <setting name="aggressiveLazyLoading" value="true"/>
  </settings>
  ```
  举个栗子
  ```
    <!--根据team的id查找player-->
    <select id="selectPlayerByTeamId" resultType="Player">
        select id,name from t_player WHERE tid=#{id}
    </select>
    
    
    <!--集合的数据来自select查询，该查询的条件是selectTeamByIdAlone查询出的id-->
    <resultMap id="teamMapAlone" type="Team">
        <id column="id" property="id"/>
        <result column="name" property="name"/>  //playerList为entity的一个字段 内容为整条sql的查询结果
        <collection property="playerList" ofType="Player" select="selectPlayerByTeamId" column="id"/>
    </resultMap>
    
    
    <select id="selectTeamByIdAlone" resultMap="teamMapAlone">
        SELECT id,name FROM t_team where id=#{id}
    </select>
  ```
    
    
## MyBatis接口和XML的映射  
   首先mapper接口名和xml的namespace一样是肯定知道的
   实现方法主要是根据JDK动态代理来实现,mybatis讲我们设置好的扫描包路径下的接口解析成代理工厂对象(MapperProxyFactory )
   之后创建接口代理类,然后判断namespace是否有同名,然后生成接口动态代理类,执行invoke方法,再执行execute执行语句

## 后记

比较拙劣
—— linyi 2019-6-5


