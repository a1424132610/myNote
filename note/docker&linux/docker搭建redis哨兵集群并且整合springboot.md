# 1、创建两个文件夹redis和sentinel文件夹用于存放docker-compose.yml文件

# 2、redis下的docker-compose.yml
```
version: "3"
services:
  master:
    image: redis:latest
    container_name: my_redis_master
    command: redis-server --requirepass root  # 在连接容器时需要密码
    ports:
      - 6379:6379
  slave1:
    image: redis:latest
    depends_on:  # 这里目的是需要先启动master，随后再启动slave节点
      - master
    container_name: my_redis_slave1
    command: redis-server --slaveof my_redis_master 6379 --requirepass root --masterauth root # 再容器启动后，通过这里命令来指定主节点ip地址
    ports:
      - 6380:6379
  slave2:
    image: redis:latest
    depends_on:
      - master
    container_name: my_redis_slave2
    ports:
     - 6381:6379
    command: redis-server --slaveof my_redis_master 6379 --requirepass root --masterauth root
networks:   # 这里是配置网络环境，目的是让容器之间能够相互连接，如果不配置，哨兵将获取不到从节点的信息，并且无法转换master节点
  default:
    external:
      name: redis_net
```

# 3、sentinel下的docker-compose.yml文件以及sentinel.conf配置文件
```
version: '3'
services:
  sentinel1:
    image: redis
    container_name: redis-sentinel-1
    networks:
      - redis_net
    ports:
      - 26379:26379
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf  # 启动哨兵，并且指定配置文件
    volumes:
      - ./sentinel1.conf:/usr/local/etc/redis/sentinel.conf

  sentinel2:
    image: redis
    container_name: redis-sentinel-2
    networks:
      - redis_net
    ports:
      - 26380:26379
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
      - ./sentinel2.conf:/usr/local/etc/redis/sentinel.conf

  sentinel3:
    image: redis
    container_name: redis-sentinel-3
    networks:
      - redis_net
    ports:
      - 26381:26379
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
      - ./sentinel3.conf:/usr/local/etc/redis/sentinel.conf
networks:  # 这里指定网络跟redis在同一个网络环境下
  redis_net:
    external:
      name: redis_net

sentinel配置文件，将下面的配置文件 cp成三分就行
port 26379
dir /tmp
sentinel monitor mymaster 192.168.16.2 6379 2  #指定集群的名字以及集群中master的ip地址，这里是容器地址，如果是用的虚拟机地址我这边会导致master转换不了。2就是指哨兵有多少个哨兵认为失效master就失效，保证超过50%就行
sentinel auth-pass mymaster root # 重点：我就是在这里忘了配置master需要密码登陆 导致一直获取不到master节点下的从信息
sentinel down-after-milliseconds mymaster 30000 #这里是指定master在失效之后多久就认为失效
sentinel parallel-syncs mymaster 1 # failover进行主备切换时最多可以有多少个slave对新的master进行同步，值越小完成failover的时间就越长，设置为1就是每次只有一个slave处于不能处理命令的状态
sentinel failover-timeout mymaster 180000 

sentinel deny-scripts-reconfig yes
```

# 4、spring boot整合redis哨兵
```
依赖
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
      <!-- 排除lettuce包，使用jedis代替-->
      <exclusions>
        <exclusion>
          <groupId>io.lettuce</groupId>
          <artifactId>lettuce-core</artifactId>
        </exclusion>
      </exclusions>
    </dependency>

    <dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <version>3.0.1</version>
    </dependency>
    
配置
spring:
  redis:
    sentinel:
      master: mymaster
      nodes: 192.168.16.5:26379,192.168.16.6:26380,192.168.16.7:26381  # 这里是连接的docker容器中哨兵的ip
    jedis:
      pool:
        max-idle: 8
        min-idle: 0
        max-wait: -1
        max-active: 8
    password: root
    
如果使用的虚拟机中的docker ip地址 先配置路由表，这样宿主机才能访问docker容器中的哨兵
route add -p 容器的网络地址 mask 子网掩码 虚拟机ip地址 切记一定要关闭虚拟机的防火墙

配置bean
@Bean
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

        // key采用String的序列化方式
        redisTemplate.setKeySerializer(stringRedisSerializer);
        // hash的key也采用String的序列化方式
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        // value序列化方式采用jackson
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        // hash的value序列化方式采用jackson
        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }
```
# 5、哨兵工作方式
### 1、特点：
- 每个哨兵进程会每秒一次向集群中的所有redis发送ping命令
- 如果距离最近的一次ping命令的时间超过了 down-after-milliseconds设置的时间，那么当前哨兵就会将起标记为主观下线（SDOWN）
- 如果Master被标记为了主观下线，那么其余的哨兵会以每秒一次的频率确认Master是否进入了主观下线状态
- 如果有足够的哨兵（配置文件中配置）进程在指定的范围内确认Master进入主观下线，则Master就会被改为客观下线（ODWON）
- 如果没有的哨兵同意Master主服务器下线，master的客观下线状态就会被移除

### 2、故障转移(Raft算法)：
- 先选择优先级最高的（slave-priority）
- 复制偏移量大的从节点（数据最新的）
- runid最小的

### 2、缺点：
- redis较难支持在线扩容，如果集群容量达到上限时在线扩容会变的很复杂
- 每个节点存入的数据都是相同的，容器造成资源的浪费
- 容易导致数据的丢失
  - redis配置文件中
  - min-slaves-to-write 1
  - min-slaves-max-lag 10  表示至少要有1个salve节点与master数据同步超过10秒，就拒绝其它的请求
- 脑裂 （网络分区变化时，master和slave出现无法通信时，sentinel会选择新的master，那么此时client一直在向之前的master发送数据，将原master进行恢复成新master的slave时，就会丢失后来发送的数据）


# 6、Redis-Cluster集群
### 1、特点：
- 节点连接是使用二进制连接的
- 节点fail是超过半数的节点检测时才会失效
- 客户端只需要连接集群中任何一个节点即可

### 2、工作方式
```
每个节点上都有slot槽的概念，取值范围是0-16383，还有一个就是cluster用于集群管理的插件。当存取的key值到达时 redis会根据crc16进行计算，然后把计算的结果对16384取余数，这样就每个key就会有对应的编号在0-16383之间的哈希槽，通过这个值跳转到对应的节点中。
redis-cluster引入了主从模式，一个主节点对应多个从节点，当超过半数的从节点ping主节点超时时，就会认为主节点宕机了。
```
### 3、缺点：
- 数据通过异步复制，不保证数据的强一致性
- 不支持多数据库空间，集群模式下只能使用1个数据库空间 0

# 、7、redis常见问题
```
1、缓存雪崩
    描述：在同一时间段，缓存集中过期。所有的请求都去查询数据库
    解决方案：
        解锁或者队列的方式防止大量线程对数据库的一次性独写
        协调redis的过期时间，随机时间
        做二级缓存，一级缓存为短期，二级缓存为长期
        依赖隔离组件为后端限流并且降级
2、缓存穿透
    描述：在redis中没有查询到并且数据库中也没有查询到，下一次进来还是会到redis中查询，就没有查询的意义了
    解决方案：
        布隆过滤器
        将空对象记录在缓存中
3、缓存击穿
    描述：查询的某个key恰好失效，刚好又有大量的并发过来，造成DB压力
    解决方案：
        通过加锁或者队列防止大量请求透过redis到DB中
        对于热点key可以无限调长
        也可以做二级缓存
4、缓存降级
    描述：访问量剧增、服务出现问题，非核心的服务影响到核心流程的性能，但是仍需要服务还是可用
5、缓存预热
    描述：先将数据直接加载到redis中，防止用户在请求时先查数据库
    解决方案：
        定时刷新缓存
6、缓存更新
    描述：维护大量缓存，自定义缓存淘汰的策略
    解决方案：定时清理过期的缓存，先判断缓存是否过期，如果过期就去数据库获取新的数据然后再更新
```