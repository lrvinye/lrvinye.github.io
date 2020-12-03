---
title: 使用Redis作为MyBatisPlus二级缓存
date: 2020-07-10 12:43:46
category: 中间件
tags: [Redis,SpringBoot,JAVA,MyBatisPlus,数据库]
---



# 开启基于MybatisPlus的二级缓存



```
mybatis-plus.configuration.cache-enabled=true
```

> 该配置项 MP 默认开启





# 配置MybatisPlusCache缓存管理



> 加入该配置，注意该类没有注入进spring容器管理

```java
@Slf4j
public class MybatisRedisCache implements Cache {

    private final Long ttl = Long.parseLong("24");//缓存失效时间，单位小时

    // 读写锁
    private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock(true);
    private  String id ;

    //这里使用了springboot的redis缓存，由于无法注入Spring容器，下面会进行代码运行时注入
    private RedisTemplate<String, Object> redisTemplate;

    public MybatisRedisCache(final String id) {
        if (id == null) {
            throw new IllegalArgumentException("Cache instances require an ID 缓存实例需要一个ID");
        }
        this.id = id;
    }

    @Override
    public String getId() {
        return this.id;
    }

    @Override
    public void putObject(Object key, Object value) {
        if (redisTemplate == null) {
            //由于启动期间注入失败，只能运行期间注入
            redisTemplate = SpringContextHolder.getBean("redisTemplate");
        }
        if (value != null) {
            redisTemplate.opsForValue().set(key.toString(), value);
            //此处修改TTL
            redisTemplate.expire(key.toString(),ttl, TimeUnit.HOURS);
        }
    }

    @Override
    public Object getObject(Object key) {
        if (redisTemplate == null) {
            //由于启动期间注入失败，只能运行期间注入
            redisTemplate = SpringContextHolder.getBean("redisTemplate");
        }
        try {
            if (key != null) {
                return redisTemplate.opsForValue().get(key.toString());
            }
        } catch (Exception e) {
            log.error("缓存出错:{}",e.getLocalizedMessage());
            e.printStackTrace();
        }
        return null;
    }

    @Override
    public Object removeObject(Object key) {
        if (redisTemplate == null) {
            //由于启动期间注入失败，只能运行期间注入
            redisTemplate = SpringContextHolder.getBean("redisTemplate");
        }
        if (key != null) {
            redisTemplate.delete(key.toString());
        }
        return null;
    }

    @Override
    public void clear() {
        log.debug("清空缓存");
        if (redisTemplate == null) {
            redisTemplate = SpringContextHolder.getBean("redisTemplate");
        }
        Set<String> keys = redisTemplate.keys("*:" + this.id + "*");
        if (!CollectionUtils.isEmpty(keys)) {
            redisTemplate.delete(keys);
        }
    }

    @Override
    public int getSize() {
        if (redisTemplate == null) {
            //由于启动期间注入失败，只能运行期间注入
            redisTemplate = SpringContextHolder.getBean("redisTemplate");
        }
        Long size = redisTemplate.execute(RedisServerCommands::dbSize);
        return size.intValue();
    }

    @Override
    public ReadWriteLock getReadWriteLock() {
        return this.readWriteLock;
    }
}
```



## SpringContextHolder  Spring容器工具类



> 注意该工具类需要注入spring容器

```java

@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext applicationContext = null;

    /**
     * 获取静态变量中的ApplicationContext.
     */
    public static ApplicationContext getApplicationContext() {
        assertContextInjected();
        return applicationContext;
    }

    /**
     * 从静态变量applicationContext中得到Bean, 自动转型为所赋值对象的类型.
     */
    public static <T> T getBean(String name) {
        assertContextInjected();
        return (T) applicationContext.getBean(name);
    }

    /**
     * 从静态变量applicationContext中得到Bean, 自动转型为所赋值对象的类型.
     */
    public static <T> T getBean(Class<T> requiredType) {
        assertContextInjected();
        return applicationContext.getBean(requiredType);
    }

    /**
     * 清除SpringContextHolder中的ApplicationContext为Null.
     */
    public static void clearHolder() {
        applicationContext = null;
    }

    /**
     * 实现ApplicationContextAware接口, 注入Context到静态变量中.
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextHolder.applicationContext = applicationContext;
    }

    /**
     * 检查ApplicationContext不为空.
     */
    private static void assertContextInjected() {
        Validate.validState(applicationContext != null,
                "applicaitonContext属性未注入, 请定义SpringContextHolder.");
    }

    /// 获取当前环境
    public static String getActiveProfile() {
        return applicationContext.getEnvironment().getActiveProfiles()[0];
    }

}
```



# Mapper类注解指定缓存配置



```java
@CacheNamespace(implementation= MybatisRedisCache.class,eviction=MybatisRedisCache.class)
public interface CommonMapper extends BaseMapper<Common> {
```





**大功告成！**