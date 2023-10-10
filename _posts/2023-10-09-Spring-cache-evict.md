---
title: 增强RedisCache中@CacheEvict实现key的模糊匹配
author: mingqing
date: 2023-10-09 19:44:00 +0800
categories: [ORM, Redis]
tags: [mybatis, cache, aop]
math: true
mermaid: true
---

`@CacheEvict`可以清除指定的 key，同时可以指定`allEntries = true`清空 namespace 下的所有元素，这也导致缺乏灵活性。

redis 的查询和删除都可以做模糊匹配，所以如何让`@CacheEvict`也支持模糊匹配清除？

## @CacheEvict 实现原理

`@CacheEvict`是通过 AOP 实现的，其中核心的类是`CacheAspectSupport`，具体的调用路径如下：

### **CacheIntercepter 工作时调用超类方法**

- AOP Alliance MethodInterceptor for declarative cache management using the common Spring caching infrastructure (org.springframework.cache.Cache).

- Derives from the CacheAspectSupport class which contains the integration with Spring's underlying caching API.

- CacheInterceptor simply calls the relevant superclass methods in the correct order.

- CacheInterceptors are **thread-safe**.

```java
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

  @Override
  @Nullable
  public Object invoke(final MethodInvocation invocation) throws Throwable {
    // ...
    try {
      return execute(aopAllianceInvoker, target, method, invocation.getArguments());
    }
  }
}
```

### **CacheAspectSupport 执行具体的增强方法**

具体的执行过程如下：

1. **同步执行逻辑**：代码检查是否需要同步执行（Synchronized Invocation），简而言之就是如果@Cacheable中满足相应的条件，则会调用线程安全的方法来从缓存中取出缓存实例（cache），否则它将直接调用底层方法（invokeOperation）；
2. **处理提前失效的缓存**（`@CacheEvict(beforeInvocation = true`）：类似于与Intercepter中的`preHandler()`，z直接调用相应的清楚缓存方法`processCacheEvicts`；
3. **检查是否存在`@Cacheable(condition/unless = '...') `和`@CachePut()`**，分为以下几种情况：
   - 有可用缓存，**且**无需存储新的缓存：直接使用缓存中数据`wrapCacheValue`作为返回值；
   - 无可用缓存，**或**需要存储新的缓存：调用底层方法（invokeOperation）获得数据作为返回值同时存入缓存；
4. **处理延迟失效的缓存**（`@CacheEvict`）

```java
@Nullable
private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
  // Special handling of synchronized invocation
  if (contexts.isSynchronized()) {
    CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
    if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
      Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
      Cache cache = context.getCaches().iterator().next();
      try {
        return wrapCacheValue(method, handleSynchronizedGet(invoker, key, cache));
      }
      catch (Cache.ValueRetrievalException ex) {
        // Directly propagate ThrowableWrapper from the invoker,
        // or potentially also an IllegalArgumentException etc.
        ReflectionUtils.rethrowRuntimeException(ex.getCause());
      }
    }
    else {
      // No caching required, only call the underlying method
      return invokeOperation(invoker);
    }
  }


  // Process any early evictions
  processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
      CacheOperationExpressionEvaluator.NO_RESULT);

  // Check if we have a cached item matching the conditions
  Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

  // Collect puts from any @Cacheable miss, if no cached item is found
  List<CachePutRequest> cachePutRequests = new ArrayList<>();
  if (cacheHit == null) {
    collectPutRequests(contexts.get(CacheableOperation.class),
        CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
  }

  Object cacheValue;
  Object returnValue;

  if (cacheHit != null && !hasCachePut(contexts)) {
    // If there are no put requests, just use the cache hit
    cacheValue = cacheHit.get();
    returnValue = wrapCacheValue(method, cacheValue);
  }
  else {
    // Invoke the method if we don't have a cache hit
    returnValue = invokeOperation(invoker);
    cacheValue = unwrapReturnValue(returnValue);
  }

  // Collect any explicit @CachePuts
  collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

  // Process any collected put requests, either from @CachePut or a @Cacheable miss
  for (CachePutRequest cachePutRequest : cachePutRequests) {
    cachePutRequest.apply(cacheValue);
  }

  // Process any late evictions
  processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

  return returnValue;
}
```

### processCacheEvicts 缓存清理

可以看到，除了一堆判断`beforeInvocaiton`和`isConditionPassing`的逻辑来从上面论述的`execute()`不同位置跳出来完成对应的`@Cacheable(conditon='...')`和`beforInvocaiton`操作，核心是通过调用`doClear()`和`doEvict()`来完成清理工作。

当`allEntries = true`时会执行`doClear()`，否则执行`doEvict()`。

```java
private void processCacheEvicts(
  Collection<CacheOperationContext> contexts, boolean beforeInvocation, @Nullable Object result) {

  for (CacheOperationContext context : contexts) {
    CacheEvictOperation operation = (CacheEvictOperation) context.metadata.operation;
    if (beforeInvocation == operation.isBeforeInvocation() && isConditionPassing(context, result)) {
      performCacheEvict(context, operation, result);
    }
  }
}

private void performCacheEvict(
  CacheOperationContext context, CacheEvictOperation operation, @Nullable Object result) {

  Object key = null;
  for (Cache cache : context.getCaches()) {
    if (operation.isCacheWide()) {
      logInvalidating(context, operation, null);
      doClear(cache, operation.isBeforeInvocation());
    }
    else {
      if (key == null) {
        key = generateKey(context, result);
      }
      logInvalidating(context, operation, key);
      doEvict(cache, key, operation.isBeforeInvocation());
    }
  }
}
```

### doEvict/doClear



抽象类`AbstractCacheInvoker`提供了相应的接口方法，最终`RedisCache`类中提供了相应的`evict`和`clear`具体实现：

```java
protected void doEvict(Cache cache, Object key, boolean immediate) {
  try {
    if (immediate) {
      cache.evictIfPresent(key);
    }
    else {
      cache.evict(key);
    }
  }
  catch (RuntimeException ex) {
    getErrorHandler().handleCacheEvictError(ex, cache, key);
  }
}

/**
 * Execute {@link Cache#clear()} on the specified {@link Cache} and
 * invoke the error handler if an exception occurs.
 */
protected void doClear(Cache cache, boolean immediate) {
  try {
    if (immediate) {
      cache.invalidate();
    }
    else {
      cache.clear();
    }
  }
  catch (RuntimeException ex) {
    getErrorHandler().handleCacheClearError(ex, cache);
  }
}
```
可以看到，最终通过cacheWriter将相应的序列化后的key和删除操作一同写出。

我们看到如果`clear()`这个方法，其实也是模糊删除的，只是他的key规则是`namespace:: *`。

```java
@Override
public void evict(Object key) {
  cacheWriter.remove(name, createAndConvertCacheKey(key));
}

@Override
public void clear() {
  byte[] pattern = conversionService.convert(createCacheKey("*"), byte[].class);
  cacheWriter.clean(name, pattern);
}

// createCacheKey构造key
protected String createCacheKey(Object key) {

  String convertedKey = convertKey(key);

  if (!cacheConfig.usePrefix()) {
    return convertedKey;
  }

  return prefixCacheKey(convertedKey);
}
// 有前缀的情况下
private String prefixCacheKey(String key) {

  // allow contextual cache names by computing the key prefix on every call.
  return cacheConfig.getKeyPrefixFor(name) + key;
}
```

## 实现自定义的evict方法

重点需要重写`RedisCache`的`evict`方法，下面新建一个`RedisCacheResolver`继承`RedisCache`，并且重写`evict`方法。

这里的通配符参考了上述`clear()`中的`convert(createCacheKey("*"), byte[].class)`，当然也可以实现其他诸如正则表达式之类的匹配逻辑。

```java
public class RedisCacheResolver extends RedisCache {

  private final String name;
  private final RedisCacheWriter cacheWriter;
  private final ConversionService conversionService;

  public RedisCacheResolver(String name, RedisCacheWriter cacheWriter,
      RedisCacheConfiguration cacheConfig) {
    super(name, cacheWriter, cacheConfig);
    this.name = name;
    this.cacheWriter = cacheWriter;
    this.conversionService = cacheConfig.getConversionService();

  }

  @Override
  public void evict(Object key) {

    if (key instanceof String) {
      String keyString = key.toString();
      // 后缀删除
      if (StringUtils.endsWithIgnoreCase(keyString, "*")) {
        evictLikeSuffix(keyString);
        return;
      }

      super.evict(key);
    }
  }

  private void evictLikeSuffix(String keyString) {
    // 后缀删除
    byte[] pattern = this.conversionService.convert(this.createCacheKey(keyString), byte[].class);
    if (pattern == null) {
      return;
    }
    this.cacheWriter.clean(this.name, pattern);
  }
}
```

## 将自定义的RedisCache加入到RedisCacheManager中

通过继承`RedisCacheManager`后重写`createRedisCache()`将之前自定义的`RedisCacheResolver`加入管理，这样实例化的`RedisCache`就是最后调用前文中自定义的`evict()`

```java
public class RedisCacheManagerResolver extends RedisCacheManager {

  private final RedisCacheWriter cacheWriter;
  private final RedisCacheConfiguration defaultCacheConfig;

  public RedisCacheManagerResolver(RedisCacheWriter cacheWriter,
      RedisCacheConfiguration defaultCacheConfiguration) {
    super(cacheWriter, defaultCacheConfiguration);
    this.cacheWriter = cacheWriter;
    this.defaultCacheConfig = defaultCacheConfiguration;
  }

  public RedisCacheManagerResolver(RedisCacheWriter cacheWriter,
      RedisCacheConfiguration defaultCacheConfiguration, String... initialCacheNames) {
    super(cacheWriter, defaultCacheConfiguration, initialCacheNames);
    this.cacheWriter = cacheWriter;
    this.defaultCacheConfig = defaultCacheConfiguration;
  }

  public RedisCacheManagerResolver(RedisCacheWriter cacheWriter,
      RedisCacheConfiguration defaultCacheConfiguration, boolean allowInFlightCacheCreation,
      String... initialCacheNames) {
    super(cacheWriter, defaultCacheConfiguration, allowInFlightCacheCreation, initialCacheNames);
    this.cacheWriter = cacheWriter;
    this.defaultCacheConfig = defaultCacheConfiguration;
  }

  public RedisCacheManagerResolver(RedisCacheWriter cacheWriter,
      RedisCacheConfiguration defaultCacheConfiguration,
      Map<String, RedisCacheConfiguration> initialCacheConfigurations) {
    super(cacheWriter, defaultCacheConfiguration, initialCacheConfigurations);
    this.cacheWriter = cacheWriter;
    this.defaultCacheConfig = defaultCacheConfiguration;
  }

  public RedisCacheManagerResolver(RedisCacheWriter cacheWriter,
      RedisCacheConfiguration defaultCacheConfiguration,
      Map<String, RedisCacheConfiguration> initialCacheConfigurations,
      boolean allowInFlightCacheCreation) {
    super(cacheWriter, defaultCacheConfiguration, initialCacheConfigurations,
        allowInFlightCacheCreation);
    this.cacheWriter = cacheWriter;
    this.defaultCacheConfig = defaultCacheConfiguration;
  }

  public RedisCacheManagerResolver(RedisConnectionFactory redisConnectionFactory,
      RedisCacheConfiguration cacheConfiguration) {
    this(RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory), cacheConfiguration);
  }


  @Override
  protected RedisCache createRedisCache(String name, RedisCacheConfiguration cacheConfig) {
    return new RedisCacheResolver(name, cacheWriter,
        cacheConfig != null ? cacheConfig : defaultCacheConfig);
  }

  @Override
  public Map<String, RedisCacheConfiguration> getCacheConfigurations() {
    Map<String, RedisCacheConfiguration> configurationMap = new HashMap<>(getCacheNames().size());
    getCacheNames().forEach(name -> {
      RedisCache cache = RedisCacheResolver.class.cast(lookupCache(name));
      configurationMap.put(name, cache != null ? cache.getCacheConfiguration() : null);
    });
    return Collections.unmodifiableMap(configurationMap);
  }
}
```

## 最后加入到IOC容器中随配置文件一同加载

这里构造的`cacheManager`中所使用的都是默认的配置文件，如有其他自定义配置也可以在实例化`redisCacheManager`中加入，比如设置ttl之类的，这里先不赘述了。

```java
@Configuration
public class RedisConfig {

	// ...

  @Bean
  public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory,
      RedisCacheConfiguration defaultCacheConfiguration,
      Map<String, RedisCacheConfiguration> cacheConfigurationMap) {
    // Create and configure a custom RedisCacheManagerResolver
    RedisCacheWriter cacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(
        redisConnectionFactory);

    return new RedisCacheManagerResolver(cacheWriter,
        defaultCacheConfiguration, cacheConfigurationMap);
  }

  @Bean
  public RedisCacheConfiguration defaultCacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig();
  }


  @Bean
  public Map<String, RedisCacheConfiguration> cacheConfigurationMap() {
    return new LinkedHashMap<>();
  }
}
```

## 使用

如下代码所示，该方法调用时就会将Redis中`user.name`开头的key删除，不至于影响其他的缓存。

```java
@PutMapping
@CacheEvict(value = "userCache", key = "#user.name + '*'")
public User update(User user) {
  userService.updateById(user);
  return user;
}
```
