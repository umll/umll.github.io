---
title: mysql和redis库存扣减优化
date: 2024-11-10 11:00:00 +0800
categories: [Redis]
tags: [Redis]

---

### 问题引出

最近在设计==校园超市小程序==的一些核心业务流程，对于订单扣减库存的问题让我思考良久，目前大多数的流程是：

1. 加购时减库存
2. 确认订单页减库存
3. 下单扣减库存
4. 支付扣减库存



> 采用jmeter进行压测，库存初始值50，线程数量1000个，1秒以内启动全部，一个线程循环2次，共2000个请求

### MySQL方案

```sql
<update id="decreaseStock">
    UPDATE stock
    SET stock_num = stock_num - 1
    WHERE product_id = #{id}
</update>
```

