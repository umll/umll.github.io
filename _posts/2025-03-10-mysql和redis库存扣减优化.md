---
title: mysql和redis库存扣减优化
date: 2025-03-10 11:00:00 +0800
categories: [Redis]
tags: [Redis]

---

### 问题引出

最近在设计==校园超市小程序==的一些核心业务流程，对于订单扣减库存的问题让我思考良久，是直接锁定MySQL的库存数据呢，还是利用缓存存储库存？



> 采用jmeter进行压测，库存初始值50，线程数量1000个，1秒以内启动全部，一个线程循环2次，共2000个请求

![img1](/assets/blog-image/blog25-3-10/img1.png)



### MySQL方案

```sql
<update id="decreaseStock">
    UPDATE product_stock
    SET product_stock = product_stock - 1
    WHERE product_id = #{id}
</update>
```

测试结果：

![img2](/assets/blog-image/blog25-3-10/img2.png)

这种情况肯定会超卖，增加**AND stock_num >= 1**条件，即可避免超卖。

```sql
<update id="decreaseStock">
    UPDATE product_stock
    SET product_stock = product_stock - 1
    WHERE product_id = #{id} AND product_stock >= 1
</update>
```

压测情况：

![img3](/assets/blog-image/blog25-3-10/img3.png)

吞吐量为171，如果系统的并发量不高，那么这种方案是可取的。

但需要注意`update`语句，若没有`where`条件索引的话，将会锁注整张表（可重复读）

如果where条件没有命中索引，那么会基于next-key lock（记录锁和间隙锁的组合）对整个表的所有记录加上这个锁，进行全表扫描，这个时候其他记录想要更新就会被阻塞。但是不一定是有了索引就不会锁住整个表，这是由优化器决定的。



### Redis方案

可以将库存放入Redis中，然后从Redis进行库存扣减。

![img4](/assets/blog-image/blog25-3-10/img4.png)

性能相比于MySQL高了很多，但直接扣减库存还是有超卖问题。

要确保Redis不超买，需要先判断当前的库存，若大于0则扣减，并且查询和扣减需要为原子性操作，这里可以使用Redis提供的`lua`脚本。

```lua
local key = KEYS[1]

-- 检查键是否存在
local exists = redis.call('EXISTS', key)
if exists == 1 then
    -- 键存在，获取值
    local value = redis.call('GET', key)
    if tonumber(value) > 0 then
        -- 如果值大于0，则递减
        redis.call('DECR', key)
        return 1  -- 表示递减成功
    else
        return 0  -- 表示递减失败，值不大于0
    end
else
    return -1  -- 表示递减失败，键不存在
end
```

使用Redis扣减库存的确能增加系统的性能，但是为了解决缓存与数据库的数据一致性问题，我们需要增加额外的逻辑去维护。

### Redis同步库存到MySQL

可以采用MQ来存储库存的同步信息，把库存的同步信息交给MQ，MQ再交到消费系统，进行减库存的操作，由MQ保证消息被消费，实现最终一致性。

```java
    public ResponseResult<String> submit() {
//        getBaseMapper().submit();
        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/decreseStock.lua")));
        redisScript.setResultType(Long.class);
        Long result = (Long) redisTemplate.execute(redisScript, Collections.singletonList("stock:1"));
        if (result == 1) {
            rocketTemplate.send("order.exchange", "order.submit", "订单提交成功");
        }
        return ResponseResult.okResult();
    }
```

利用lua脚本的原子性，保障库存安全，并且利用MQ保证高并发下快速响应。

但是事需要把库存的信息保存到Redis，并保证Redis 和 Mysql 数据同步。缺点是redis宕机后不能下单。

同时，这种方案还需要考虑MQ消息发送失败、或者MySQL库存扣减失败，并且实际情况还有订单的生成和库存之间的一致性也要考虑。

### 总结

在高并发情况下，采用Redis+MQ+MySQL，下单和实际库存扣减异步执行。

在并发情况不高，平常商品或者正常购买流程，可以采用MySQL数据库乐观锁的处理。