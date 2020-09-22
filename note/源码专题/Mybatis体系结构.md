# 一、传统jdbc的弊端
- 1、没有使用连接池，需要频繁创建和关联链接，消耗大
- 2、一旦经过代码修改，需要整体编译，不利于系统维护
- 3、预编译，需要设置index，不利于维护
- 4、返回的result结果集也需要硬编码

# 二、Mybatis
### #和$符号的区别
> 相同点：都是对参数进行编译
> #预编译，防止sql注入
> $字符串拼接的方式

# 三、Mybatis源码分析(本质：以对象的方式去操作数据库)
- Configuration  #管理xml的全局配置关系类
- SqlSessionFactory # Session管理工厂接口
- Session # SqlSession提供了操作数据库的方法
- Executor # 执行器接口，SqlSession通过执行器操作数据库
- MappedStatement # 底层封装对象，对操作数据库存储封装，包括sql语句，输入输出
- StatementHandler # 具体操作数据库相关的handler接口
- ResultSetHandler # 具体操作数据库返回结果的handler接口
- TypeHandler      # 类型处理器

```
解析过程
org.apache.ibatis.session.SqlSessionFactoryBuilder  #使用bulder获取到SqlSessionFactory
返回一个configuration对象，其中有Enviroment对象里面包含了事务管理工厂以及数据库信息
org.apache.ibatis.builder.xml.XMLConfigBuilder#parse   #调用parse来进行解析xml文件
org.apache.ibatis.builder.xml.XMLConfigBuilder#parseConfiguration  #解析对应节点
org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#DefaultSqlSessionFactory  #调用当前类的构造方法，并且将configuration设置进去
org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#openSessionFromDataSource #开启session，获取到事务管理器
org.apache.ibatis.session.Configuration#newExecutor(org.apache.ibatis.transaction.Transaction, org.apache.ibatis.session.ExecutorType)             #获取到执行器，默认为SimpleExecutor
org.apache.ibatis.executor.CachingExecutor          #会将上面默认的执行器存入到当中。一级缓存，自动缓存。key:id+sql+limit+offset。缓存是会话级别
org.apache.ibatis.plugin.InterceptorChain#pluginAll     # 会遍历执行拦截器中的plugin方法，责任链模式
org.apache.ibatis.plugin.Plugin#wrap       #调用wrap方法生成动态代理类

一级缓存，自动缓存。key:id+sql+limit+offset。缓存是会话级别
二级缓存，全局缓存，需要单独开启（第三方实现）

操作数据库
org.apache.ibatis.session.defaults.DefaultSqlSession#selectList(java.lang.String) #调用查询方法
org.apache.ibatis.session.Configuration#getMappedStatement(java.lang.String)  # 获取到xml中对应的方法名称
org.apache.ibatis.executor.CachingExecutor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler) # 调用Executor中的query方法
org.apache.ibatis.mapping.MappedStatement#getBoundSql               #获取到xml当中写的对应的sql语句
org.apache.ibatis.executor.BaseExecutor#createCacheKey              #自动创建缓存
org.apache.ibatis.executor.CachingExecutor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler, org.apache.ibatis.cache.CacheKey, org.apache.ibatis.mapping.BoundSql)                                 #再次调用query方法
org.apache.ibatis.mapping.MappedStatement#getCache                  #调用getCache()方法获取缓存如果没有过去到再查数据库
org.apache.ibatis.executor.BaseExecutor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler, org.apache.ibatis.cache.CacheKey, org.apache.ibatis.mapping.BoundSql)
org.apache.ibatis.executor.BaseExecutor#queryFromDatabase           #查找数据库
org.apache.ibatis.executor.resultset.ResultSetHandler#handleResultSets   #处理返回的结果集

调用getMapper()方法
org.apache.ibatis.session.Configuration#getMapper
org.apache.ibatis.binding.MapperRegistry#getMapper   #根据限定名，去map当中获取
org.apache.ibatis.binding.MapperProxyFactory#newInstance(org.apache.ibatis.session.SqlSession)  #创建动态代理对象
org.apache.ibatis.binding.MapperProxy       #创建代理对象
org.apache.ibatis.binding.MapperMethod#execute   #其中进行判定是查找还是删除更新等操作
```
### 解析xml中为mapper路径时解析过程
```
org.apache.ibatis.session.Configuration#addMappers(java.lang.String)   #如果类型是为包路径时
org.apache.ibatis.binding.MapperRegistry#addMappers(java.lang.String, java.lang.Class<?>)   #获取当前包路径下的所有类
org.apache.ibatis.binding.MapperRegistry#addMapper   #判断当前类是否是接口，将其push到map当中保存，并且new一个代理类的工厂对象
org.apache.ibatis.builder.annotation.MapperAnnotationBuilder#parse  #调用parse去解析类中的方法
org.apache.ibatis.builder.annotation.MapperAnnotationBuilder#parseStatement  #解析对应的注解
org.apache.ibatis.builder.MapperBuilderAssistant#addMappedStatement(java.lang.String, org.apache.ibatis.mapping.SqlSource, org.apache.ibatis.mapping.StatementType, org.apache.ibatis.mapping.SqlCommandType, java.lang.Integer, java.lang.Integer, java.lang.String, java.lang.Class<?>, java.lang.String, java.lang.Class<?>, org.apache.ibatis.mapping.ResultSetType, boolean, boolean, boolean, org.apache.ibatis.executor.keygen.KeyGenerator, java.lang.String, java.lang.String, java.lang.String, org.apache.ibatis.scripting.LanguageDriver, java.lang.String) #调用当前方法，将其添加到Configuration中的MappedStatement map中保存
```