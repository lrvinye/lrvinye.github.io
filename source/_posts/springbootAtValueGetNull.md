---
title: 【已解决】Spring boot @Value 获取值为 null
category: '错误解决办法'
date: 2020-06-04 19:46:48
tags: [SpringBoot,JAVA,已解决]
---

- [x] **是否解决**



## 环境or场景

**环境**

- springboot 2.x
- JB idea



**场景**

调试微信支付服务端

通过以下方式

```
@Value("${wxpay.apiclientCertPath}"
private  String path;
```

在类构造方法中调用

```
public WXConfigUtil() throws Exception {
        File file = new File(path);
        InputStream certStream = new FileInputStream(file);
        this.certData = new byte[(int) file.length()];
        certStream.read(this.certData);
        certStream.close();
    }
```



## 错误内容

```
Caused by: java.lang.NullPointerException: null
	at java.base/java.io.File.<init>(File.java:278) ~[na:na]
	at com.cherryez.ms.pay.config.WXConfigUtil.<init>(WXConfigUtil.java:41) ~[classes/:na]
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[na:na]
	at java.base/java.lang.reflect.Constructor.newInstance(Constructor.java:490) ~[na:na]
	at org.springframework.beans.BeanUtils.instantiateClass(BeanUtils.java:204) ~[spring-beans-5.2.6.RELEASE.jar:5.2.6.RELEASE]
	... 19 common frames omitted
```

经过多次调试

错误内容的这一行找到源码，

```
at java.base/java.io.File.<init>(File.java:278) ~[na:na]
```

发现是传入了一个`null`值到 new File() 里面了



## 可能原因

往常来看，按上面的@Value 的使用方法都是可以奏效的

但是这次由于 这个参数 `path` 是在类构造方法中使用

猜测可能是由于 spring bean中的加载顺序导致的

也就是说 ：在bean创建时运行到构造方法的时候，还没有读取配置文件中的自定义配置（springboot的配置项会在bean创建前加载，但是自定义的配置可能不会）

这里的 `@Value("${wxpay.apiclientCertPath}"`  就是我们的一个自定义配置项



## 解决方法

使用这样的写法

```
public WXConfigUtil(@Value("${wxpay.apiclientCertPath}") String path) throws Exception {
        File file = new File(path);
        InputStream certStream = new FileInputStream(file);
        this.certData = new byte[(int) file.length()];
        certStream.read(this.certData);
        certStream.close();
    }
```

可以在构造方法的时候就提前加载自定义的配置项

从而取到一个 非null 的值

## 总结or心得



**总结一下 @value 能获取到值的要求**

1. 使用@value 的类中必须被`@Service 或@Component`注解
2. 从请求进入接口开始，所有的方法都必须注入到 Spring boot容器中，被Spring boot所管理。
3. 对象必须使用@Autowired注入，才能正常使用@Value注解,(而不是new 的对象)

