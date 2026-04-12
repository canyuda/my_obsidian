# Redisson 缓存规范

## Cache-Aside 模式（读）

```java
public UserVO getUser(Long userId) {
    String key = "user:info:" + userId;
    RBucket<UserVO> bucket = redisson.getBucket(key);
    
    // 1. 读缓存
    UserVO cached = bucket.get();
    if (cached != null) return cached;
    
    // 2. 防击穿：分布式锁
    RLock lock = redisson.getLock("lock:user:" + userId);
    try {
        if (!lock.tryLock(5, 10, TimeUnit.SECONDS)) {
            throw new BizException(1001, "系统繁忙");
        }
        
        // 双重检查
        cached = bucket.get();
        if (cached != null) return cached;
        
        // 3. 读DB
        UserVO db = userMapper.selectById(userId);
        if (db == null) {
            // 缓存空值，防穿透（TTL=5分钟）
            bucket.set(null, 5, TimeUnit.MINUTES);
            return null;
        }
        
        bucket.set(db, 30, TimeUnit.MINUTES);  // 正常缓存30分钟
        return db;
    } finally {
        lock.unlock();
    }
}
```

## 写操作（延迟双删）

```java
public void updateUser(UserDTO dto) {
    String key = "user:info:" + dto.getId();
    
    // 1. 删缓存
    redisson.getBucket(key).delete();
    
    // 2. 更新DB
    userMapper.updateById(dto);
    
    // 3. 延迟双删（异步500ms后再次删除）
    CompletableFuture.delayedExecutor(500, TimeUnit.MILLISECONDS)
        .execute(() -> redisson.getBucket(key).delete());
}
```

## 分布式锁（RedissonLock）

```java
/**
 * 锁自动续期（看门狗），默认30秒，业务完成自动释放
 */
public void deductStock(Long skuId, Integer num) {
    String lockKey = "lock:inventory:" + skuId;
    RLock lock = redisson.getLock(lockKey);
    
    try {
        // 等待10秒，锁持有30秒（自动续期）
        boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
        if (!acquired) {
            throw new BizException(3001, "库存并发竞争失败");
        }
        
        // 业务逻辑...
        doDeduct(skuId, num);
        
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        // 必须判断当前线程持有才解锁
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

## Key 命名规范
- 缓存：`{业务}:{对象}:{ID}`，如 `user:info:10001`
- 锁：`lock:{业务}:{资源}`，如 `lock:inventory:sku_123`
- 幂等：`mq:consumed:{msgId}`
- TTL：热点数据30分钟，普通数据60分钟，空值5分钟
