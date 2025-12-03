### 方案一：多个单表缓存，通过关联字段进行查询拼接，需要 n 次实现多表联查

缓存同步实现逻辑：
多表联查的结果 = 实时查询多个单表缓存 + 应用层拼接。所以：
单表字段更新：同步更新该表的 Redis 缓存（如 HMSET t_user:123 user_name "新名称"），后续联查自然拿到新值；
单表数据删除：同步删除该表的 Redis 缓存（如 DEL t_user:123），后续联查查不到该单表缓存，自然不会拼接出旧数据（实现 “删除同步”）。

### Redis 中 “视图” 方案

本质：（类似 MySQL 视图的聚合效果），单表更新时同步多表缓存的核心是：通过 “反向索引” 精准定位所有依赖该单表的视图缓存，批量删除后由后续查询自动重建最新视图—— 全程无模糊匹配，性能可控。

同步核心逻辑（3 步走）：
单表更新 → 精准找关联视图（通过反向索引） → 批量删除视图缓存（后续查询自动重建最新视图）（不直接更新视图缓存：一个单表可能关联多个视图，删除重建比逐个更新更高效）

完整实现方案（示例：用户 - 订单 - 店铺视图）

1. 初始化：创建视图 + 反向索引
   创建视图时，同步维护 “单表 → 视图” 的反向索引（用 Set 存储），确保后续能精准定位。
   Redis 命令示例：

```
# 1. 创建单表缓存（基础数据）
HMSET t_user:123 user_id "123" user_name "张三"
HMSET t_order:456 order_id "456" user_id "123" shop_id "789"
HMSET t_shop:789 shop_id "789" shop_name "便利店"

# 2. 创建视图缓存（聚合多表数据，模拟 MySQL 视图）
HMSET view:user_order_shop:123_456
  user_id "123" user_name "张三"
  order_id "456" order_status "2"
  shop_id "789" shop_name "便利店"
EXPIRE view:user_order_shop:123_456 86400  # 视图过期时间（兜底）

# 3. 维护反向索引（核心：记录哪些视图依赖该单表）
# - 用户123对应的所有视图
SADD index:t_user:view:123 view:user_order_shop:123_456
# - 订单456对应的所有视图
SADD index:t_order:view:456 view:user_order_shop:123_456
# - 店铺789对应的所有视图
SADD index:t_shop:view:789 view:user_order_shop:123_456
```

2. 单表更新：同步视图缓存（精准删除）
   以 “更新用户 123 的名称” 为例，同步所有包含该用户的视图缓存：
   同步步骤（Redis 命令）：

```
# 步骤1：更新单表自身缓存（确保单表数据最新）
HMSET t_user:123 user_name "张三更新"

# 步骤2：通过反向索引，精准获取所有依赖该用户的视图缓存 Key
SMEMBERS index:t_user:view:123 → 结果：view:user_order_shop:123_456

# 步骤3：批量删除这些视图缓存（后续查询自动重建最新视图）
DEL view:user_order_shop:123_456

# 步骤4（可选）：清理反向索引（视图已删除，索引无用）
SREM index:t_user:view:123 view:user_order_shop:123_456
SREM index:t_order:view:456 view:user_order_shop:123_456
SREM index:t_shop:view:789 view:user_order_shop:123_456
```

3. 后续查询：自动重建最新视图
   当用户再次查询 “用户 123 + 订单 456 的视图” 时，应用层逻辑自动触发重建：
   3.1. 查单表缓存：t_user:123（最新名称）、t_order:456、t_shop:789；
   3.2. 聚合数据，重建视图缓存：HMSET view:user_order_shop:123_456 ...；
   3.3. 重新维护反向索引：SADD index:t_user:view:123 view:user_order_shop:123_456。

关键：反向索引的维护细节
反向索引是精准同步的核心，必须在 “视图创建 / 删除” 时同步维护：
操作 反向索引维护动作

1. 创建视图 给视图依赖的所有单表，添加 “视图 Key→ 单表索引 Set”
2. 删除视图（主动） 从所有依赖单表的索引 Set 中，移除该视图 Key
3. 视图过期（被动） 无需手动维护，索引 Set 设置 30 分钟过期（比视图长），自动清理
4. 单表删除 通过索引 Set 删除所有依赖该单表的视图，再删除单表缓存

为什么不直接更新视图缓存？
处理方式 操作成本 性能 适用场景

1. 精准删除 + 重建 1 次删除 + 后续 1 次重建 高（读多写少） 视图查询频繁
2. 直接更新视图 遍历所有关联视图逐个更新 低（写多读少） 视图更新频繁
