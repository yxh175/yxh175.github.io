---
title: 项目总结：电子商城面试必备
date: 2023-10-10 19:16:02
categories:
- 总结
tags:
- 电子商城
---

好久之前做的简单电子商城项目，进行一下回顾，并试着复述一下项目的难点与亮点

这里有针对这些问题的demo，[电子商城亮点demo](https://github.com/yxh175/web-small-demo)

## 简单介绍一下你的项目的亮点和难点

电子商城项目是我用go尝试开发的一个简易项目，其中项目主要用到了Gin做路由分布，Gorm做数据库查询。同时使用Redis实现一些相关功能。

### 项目亮点

- 权限控制及Token续约：使用Gin及jwt-token做权限拦截，并通过双token实现安全性和续期的双重保障
- 多级缓存：使用redis做缓存缓解数据库读写压力，同时通过合适的算法保证缓存一致性
- 防止超卖：使用redis做分布式锁，防止出现高并发情况下的超卖现象
- ELK日志监控：通过定制化 Zap 实现多输出源，同时将日志输出到 Console （Standard IO） 与 MQ 中，再配置 Logstash Input 使其读取 MQ 中的日志并写入 ES 中，最后在 Kibana 中展示。

### 权限拦截及双Token的实现

权限拦截主要通过Gin框架实现

通过Gin框架的中间件模式，可以实现基本的鉴权功能

路由函数可以使用`r.use`或者使用`r.group`，都可以。

```go

// 使用鉴权中间件
r.Use(middleware.JWTMiddleware())

```

鉴权中间件如何实现的呢？

首先先构建jwt-token结构体，然后写两个方法，一个分发，一个验证。

```go
var jwtSecret = []byte("secret")

type JWTClaims struct {
	ID       uint   `json:"id"`
	UserName string `json:"user_name"`
	jwt.StandardClaims
}

// GenerateToken 签发用户Token
func GenerateToken(id uint, username string) (accessToken, refreshToken string, err error) {
    ...
}

// ParseToken 验证用户token
func ParseToken(token string) (*JWTClaims, error) {
	...
}

func ParseRefreshToken(aToken, rToken string) (newAToken, newRToken string, err error) {
	...
}

```

使用jwt-token，它的核心部分是获取token，然后对token做验证，正常就`c.Next()`放行，否则就要`c.Abort()`拦截

```go
func JWTMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		code := 200
		accessToken := c.GetHeader("access-Token")
        refreshToken := c.GetHeader("refresh-Token")
		
        // 1.检验token是否有效
		if accessToken == "" {
			code = 401
			c.JSON(200, gin.H{
				"status": code,
				"data":   "token不能为空",
			})
			c.Abort()
			return
		}
        accessClaims, err := util.ParseToken(accessToken)
		if err != nil {
			code = 401
			c.JSON(200, gin.H{
				"status": code,
				"data":   "token异常",
			})
			c.Abort()
			return
		}
        // 2.检验accessToken是否过期

        if accessClaims.ExpiresAt > time.Now().Unix() {
			c.Set("id", accessClaims.ID)
			c.Set("username", accessClaims.UserName)
			c.Next()
			return
		}

        // 3. 双token续约机制


		c.Set("id", claims.ID)
		c.Set("username", claims.UserName)
		c.Next()
	}
}
```

最后在运行时，可以发现token是会检验的。当accessToken过期时，会根据refreshToken情况决定是否更新Token，更新的Token返回给前端

#### 双Token的优点

- accessToken的存在，保证了登录态的正常验证，因其过期时间的短暂也保证了帐号的安全性
- refreshToken的存在，保证了用户（即使是非活跃用户）无需在短时间内进行反复的登陆操作来保证登录态的有效性，同时也保证了活跃用户的登录态可以一直存续而不需要进行重新登录，其反复刷新也防止别人获取refreshToken干坏事

流程图如下

![双token验证流程](https://s2.loli.net/2023/10/10/HEqg1vRsxGNJu37.png)

#### 无效的Token处理

refreshToken的无效token会生成的比较多，如何处理呢？

- 简单的从浏览器移除
- 制作一个token黑名单

### 多级缓存如何实现

首先通过`demo.sql`建一个表

```sql
CREATE SCHEMA cache_demo;
use cache_demo;
DROP TABLE if exists products;

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    p_id Int Not NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    count INT NOT NULL
);
```

本模块在`cache_demo`中实现

首先先做好基本的DB查询。这里DB使用gorm，同时使用连接池和读写分离保证高性能。

```go
var _db *gorm.DB

func InitMySQL() {
	// 假装四个读写mysql服务器，进行读写分离, 电商读多写少
	read1_dsn := "root:1234@tcp(localhost:3306)/cache_demo?parseTime=true"
	read2_dsn := "root:1234@tcp(localhost:3306)/cache_demo?parseTime=true"
	write1_dsn := "root:1234@tcp(localhost:3306)/cache_demo?parseTime=true"
	write2_dsn := "root:1234@tcp(localhost:3306)/cache_demo?parseTime=true"

	db, err := gorm.Open(mysql.Open(write1_dsn), &gorm.Config{})
	if err != nil {
		panic(err)
	}
	_db = db
    _db.Use(
        dbresolver.Register(dbresolver.Config{
            Sources:  []gorm.Dialector{mysql.Open(write2_dsn)},
            Replicas: []gorm.Dialector{mysql.Open(read1_dsn), mysql.Open(read2_dsn)},
            Policy:   dbresolver.RandomPolicy{},
            // print sources/replicas mode in logger
            TraceResolverMode: true,
        }).
            SetConnMaxIdleTime(time.Hour).
            SetConnMaxLifetime(24 * time.Hour).
            SetMaxIdleConns(100).
            SetMaxOpenConns(200),
    )

	_db = _db.Set("gorm:table_options", "charset=utf8mb4")
}

func NewDBClient(ctx context.Context) *gorm.DB {
	db := _db
	return db.WithContext(ctx)
}
```

基本的增删改查

```go
type ProductDao struct {
	*gorm.DB
}

func NewProductDao(ctx context.Context) *ProductDao {
	return &ProductDao{NewDBClient(ctx)}
}

func NewProductDaoByDB(db *gorm.DB) *ProductDao {
	return &ProductDao{db}
}

// 获取商品
func (dao *ProductDao) GetProduct(id uint) (product *model.Product, err error) {
	...
}

// CreateProduct 创建商品
func (dao *ProductDao) CreateProduct(product *model.Product) error {
	...
}

// DeleteProduct 删除商品
func (dao *ProductDao) DeleteProduct(id uint) error {
	...
}

// UpdateProduct 更新商品
func (dao *ProductDao) UpdateProduct(id uint, product *model.Product) error {
	...
}

```

通过product_test.go进行测试发现结果符合预期。接下来需要增加基本的缓存。

首先增加本地缓存

`local_cache.go`

```go
// 缓存数据的结构
type Cache struct {
	mu      sync.RWMutex
	data    map[string]interface{}
	expires map[string]time.Time
}

// 创建一个新的缓存
func NewCache() *Cache {
	return &Cache{
		data:    make(map[string]interface{}),
		expires: make(map[string]time.Time),
	}
}

// 向缓存中添加数据
func (c *Cache) Set(key string, value interface{}, expiration time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = value
	c.expires[key] = time.Now().Add(expiration)
}

// 从缓存中获取数据
func (c *Cache) Get(key string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	value, ok := c.data[key]
	if !ok {
		return nil, false
	}
	expiration, exists := c.expires[key]
	if !exists || time.Now().Before(expiration) {
		return value, true
	}
	// 数据已过期，从缓存中删除
	delete(c.data, key)
	delete(c.expires, key)
	return nil, false
}

```

在`local_cache_test.go`中进行测试，可以看到cache缓存的执行时间远小于DB查询

```text
--------普通DB查询-------
运行时长：547.3µs
&{12 2 DB测试 123 33}
-----------------------


-----有缓存DB第一查询-----
运行时长：519.8µs
&{6 1 缓存测试 123 33}
-----------------------


-----有缓存缓存查询-----
运行时长：0s
&{6 1 缓存测试 123 33}
-----------------------
```

当然，验证完了查询问题，还需要设计缓存一致性

缓存一致性是业务层面的设计。所以在业务代码中进行书写。接下来加入`redis_cache`

`redis_cache.go`

```go
type RedisCache struct {
	client *redis.Client
}

// NewRedisCache 创建一个新的 Redis 缓存实例
func NewRedisCache() *RedisCache {
	return &RedisCache{
		client: redis.NewClient(&redis.Options{
			Addr:     "localhost:6379", // Redis 服务器地址
			Password: "",               // 如果有密码，设置密码
			DB:       0,                // 默认数据库
			PoolSize: 1000,
		}),
	}
}

// Ping 用于测试 Redis 连接
func (rc *RedisCache) Ping() error {
	ctx := context.Background()
	pong, err := rc.client.Ping(ctx).Result()
	if err != nil {
		return fmt.Errorf("无法连接到 Redis: %v", err)
	}
	fmt.Println("Redis 连接成功:", pong)
	return nil
}

// Set 用于设置缓存数据
func (rc *RedisCache) Set(key, value string, expiration time.Duration) error {
	ctx := context.Background()
	err := rc.client.Set(ctx, key, value, expiration).Err()
	if err != nil {
		return fmt.Errorf("设置缓存失败: %v", err)
	}
	return nil
}

// Get 用于获取缓存数据
func (rc *RedisCache) Get(key string) (string, error) {
	ctx := context.Background()
	cacheValue, err := rc.client.Get(ctx, key).Result()
	if err == redis.Nil {
		return "", err
	} else if err != nil {
		return "", fmt.Errorf("获取缓存失败: %v", err)
	}
	return cacheValue, nil
}

// Delete 用于删除缓存数据
func (rc *RedisCache) Delete(key string) error {
	ctx := context.Background()
	err := rc.client.Del(ctx, key).Err()
	if err != nil {
		return fmt.Errorf("删除缓存失败: %v", err)
	}
	return nil
}
```

使用测试函数测试，发现结果符合预期

```text
--------普通DB查询-------
运行时长：642.4µs
&{12 2 DB测试 123 33}
-----------------------


-----有缓存DB第一查询-----
运行时长：306.4407ms
&{6 1 缓存测试 123 33}
-----------------------


-----有缓存缓存查询-----
运行时长：0s
&{6 1 缓存测试 123 33}
-----------------------
```

那么在业务中如何去写呢，简单的写一下`service`模块

首先实现一下GetData模块

`service/product.go`

```go
var ProductSrvIns *ProductSrv
var ProductSrvOnce sync.Once
var key string = "product:"

type ProductSrv struct {
	localCache *cache.Cache
	redisCache *cache.RedisCache
}

func GetProductSrv() *ProductSrv {
	ProductSrvOnce.Do(func() {
		ProductSrvIns = &ProductSrv{
			localCache: cache.LocalCache,
			redisCache: cache.RDCache,
		}
	})
	return ProductSrvIns
}

func (ps *ProductSrv) GetData(c *gin.Context, pId uint) (product *model.Product, err error) {
	uniqueKey := key + fmt.Sprint(pId)
	// 查本地缓存
	if value, ok := ps.localCache.Get(uniqueKey); ok {
		// 一级缓存查询有值
		product = &model.Product{}
		err = json.Unmarshal(value.([]byte), product)
		if err != nil {
			fmt.Println("反序列化失败:", err)
			return
		}
		return
	}

	// 否则查二级缓存
	value, err := ps.redisCache.Get(uniqueKey)

	// 查询异常
	if err != nil {
		if err != redis.Nil {
			return
		}
		// 查询无果
		// 查询数据库中的数据
		product, err = dao.NewProductDao(c).GetProduct(pId)
		if err != nil {
			if err == gorm.ErrRecordNotFound {
				// 查询无果
				return &model.Product{}, nil
			} else {
				return
			}
		}
		var jsonData []byte
		jsonData, err = json.Marshal(*product)
		if err != nil {
			fmt.Println("序列化失败:", err)
			return
		}
		// 更新缓存
		ps.redisCache.Set(uniqueKey, string(jsonData), 60*time.Second)
		ps.localCache.Set(uniqueKey, string(jsonData), 10*time.Second)
		return
	}

	product = &model.Product{}
	jsonData := []byte(value)
	err = json.Unmarshal(jsonData, product)
	if err != nil {
		fmt.Println("反序列化失败:", err)
		return
	}
	return
}
```

同时实现增删改三个部分

为了简单实现，把表单需要传输的数据通过随机数实现

`service/product.go`

```go
func (ps *ProductSrv) UpdateData(c *gin.Context, pId uint, newPrice float64) (err error) {
	// 先操作数据库，在删除缓存是比较好的选择
	// 后删除防止别的线程进入数据库查询
	uniqueKey := key + fmt.Sprint(pId)
	// 更新数据库
	if err = dao.NewProductDao(c).UpdateProduct(pId, &model.Product{Price: newPrice}); err != nil {
		return
	}

	// 更新缓存, 从下到上
	err = ps.redisCache.Delete(uniqueKey)
	if err == redis.Nil {
		return nil
	}
	ps.localCache.Delete(uniqueKey)
	return
}

func (ps *ProductSrv) DeleteData(c *gin.Context, pId uint) (err error) {
	uniqueKey := key + fmt.Sprint(pId)

	if err = dao.NewProductDao(c).DeleteProduct(pId); err != nil {
		return
	}

	// 更新缓存, 从下到上
	err = ps.redisCache.Delete(uniqueKey)
	if err == redis.Nil {
		return nil
	}
	ps.localCache.Delete(uniqueKey)
	return
}

func (ps *ProductSrv) CreateData(c *gin.Context) (err error) {
	rand.Seed(time.Now().Unix())
	pid := rand.Intn(10000)
	uniqueKey := key + fmt.Sprint(pid)
	newProduct := &model.Product{
		PID:  uint(pid),
		Name: "测试",
	}
	if err = dao.NewProductDao(c).CreateProduct(newProduct); err != nil {
		return
	}
	data, err := json.Marshal(newProduct)
	if err != nil {
		return err
	}
	err = ps.redisCache.Set(uniqueKey, string(data), 60*time.Second)
	if err != nil {
		return
	}
	ps.localCache.Set(uniqueKey, data, 10*time.Second)
	return
}

```

当然，完成了基本业务实现，需要去考虑一些缓存会出现的问题，也就是业务难点

#### 缓存雪崩

因为大量的缓存数据在同一时间过期失效，或者redis宕机，导致大量的访问请求打到数据库层。使得数据库阻塞影响业务进度。

上面已经通过二级缓存缓解了这种情况，还可以使用过期时间随机化去解决。

#### 缓存击穿

缓存击穿问题也叫热点Key问题，就是一个被高并发访问并且缓存重建业务较复杂的key突然失效了，大量的请求访问会在瞬间给数据库带来巨大的冲击。

这里可以使用互斥锁和逻辑过期去实现

- 互斥锁实现简单，但是高并发情况下会发生阻塞

![互斥锁实现](https://s2.loli.net/2023/10/12/DbsjJFXN5nIYERk.png)

- 逻辑过期实现有点难度，且对redis有一定的内存占用。同时也容易发生脏读

![逻辑过期](https://s2.loli.net/2023/10/11/eTn7fdLN8iVYlBj.png)

使用逻辑过期的用户体验比较好一点，但是会对内存占用比较高，也会发生脏读的现象。

本次demo准备使用二级缓存，所以可以两个同时使用

对第一层使用逻辑过期，第二层使用互斥锁实现（目前仅实现互斥锁）

在service/Product.go中增加互斥锁相关

```go
func (ps *ProductSrv) GetData(c *gin.Context, pId uint) (product *model.Product, err error) {
	uniqueKey := key + fmt.Sprint(pId)
	// 模拟requestId
	rand.Seed(time.Now().Unix())
	requestId := fmt.Sprint(rand.Intn(10000000))
	// 查本地缓存
	if value, ok := ps.localCache.Get(uniqueKey); ok {
		// 一级缓存查询有值
		product = &model.Product{}
		err = json.Unmarshal(value.([]byte), product)
		if err != nil {
			fmt.Println("反序列化失败:", err)
			return
		}
		return
	}

	// 否则查二级缓存
	value, err := ps.redisCache.Get(uniqueKey)

	// 查询异常
	if err != nil {
		if err != redis.Nil {
			return
		}
		// 查询无果
		// 查询数据库中的数据
		// redis 上锁
		var locked bool
		locked, err = ps.redisCache.SetNx(c, lockKey, requestId)
		defer ps.redisCache.Unlock(c, lockKey, requestId)
		if err != nil {
			return
		}
		if !locked {
			time.Sleep(50 * time.Millisecond)
			return ps.GetData(c, pId)
		}
		product, err = dao.NewProductDao(c).GetProduct(pId)
		if err != nil {
			if err == gorm.ErrRecordNotFound {
				// 查询无果
				return &model.Product{}, nil
			} else {
				return
			}
		}
		var jsonData []byte
		jsonData, err = json.Marshal(*product)
		if err != nil {
			fmt.Println("序列化失败:", err)
			return
		}
		// 更新缓存
		ps.redisCache.Set(uniqueKey, string(jsonData), 60*time.Second)
		ps.localCache.Set(uniqueKey, string(jsonData), 10*time.Second)
		return
	}

	product = &model.Product{}
	jsonData := []byte(value)
	err = json.Unmarshal(jsonData, product)
	if err != nil {
		fmt.Println("反序列化失败:", err)
		return
	}
	return
}

```

这里setNx+EX实际还能增加一个守护进程来进行锁的续期，但是我没实现，有读者有能力可以尝试去实现一下。对于用redis实现互斥锁的相关问题，建议大家看一下这篇文章[Go实战：redis分布式锁开发问题](http://xiaoheinotes.com/2023/09/25/redis-distributed-locks/)

#### 缓存穿透

缓存穿透是整个查询流程都被使用，一般是被黑客攻击的常用手段。

常用的解决手段有

- 缓存null值
- 布隆过滤
- 增强id的复杂度，避免被猜测id规律
- 做好数据的基础格式校验
- 加强用户权限校验
- 做好热点参数的限流

对于上面两个，可以两个都用，缓存null值比较好实现，只需要在mysql查为空值的时候进行赋值即可。

```go
    product, err = dao.NewProductDao(c).GetProduct(pId)
    if err != nil {
        if err == gorm.ErrRecordNotFound {
            // 查询无果
            // 防止缓存穿透
            ps.localCache.Set(uniqueKey, "", 10*time.Second)
            ps.redisCache.Set(c, uniqueKey, "", 60*time.Second)
            return &model.Product{}, nil
        } else {
            return
        }
    }
```

这里就相当于是一个null值设置

同时实现布隆过滤，减少缓存的压力

```go
    import "github.com/bits-and-blooms/bloom"

    ...

	filter := bloom.NewWithEstimates(1000000, 0.01)

	if !filter.Test([]byte(fmt.Sprint(pId))) {
		return
	}

```

这里加上了后其他增加的就要做适当的调整

```go
	n1 := make([]byte, 8)
	binary.BigEndian.PutUint64(n1, uint64(pid))
	ps.filter.Add(n1)
```

### redis锁实现超卖避免

超卖的问题一种经典的并发问题，因为多个线程同时再判断库存足够的时候，扣除库存时的这段时间并不是瞬时完成的，而这个问题也导致了超卖的问题。

![超卖情况](https://s2.loli.net/2023/10/12/dIo2iNFw5LK81Xv.png)

假设线程1过来查询库存，判断出来库存大于1，正准备去扣减库存，但是还没有来得及去扣减，此时线程2过来，线程2也去查询库存，发现这个数量一定也大于1，那么这两个线程都会去扣减库存，最终多个线程相当于一起去扣减库存，此时就会出现库存的超卖问题。

对于超卖的解决有很多方法，此前做过一些解决超卖的demo测试[超卖解决demo](https://github.com/yxh175/shop_demo)

- 悲观锁

悲观锁可以实现对于数据的串行化执行，比如通过信号量，和lock都是悲观锁的代表，同时，悲观锁中又可以再细分为公平锁，非公平锁，可重入锁，等等

- 乐观锁

乐观锁会有一个版本号，每次操作数据会对版本号+1，再提交回数据时，会去校验是否比之前的版本大1 ，如果大1 ，则进行操作成功，这套机制的核心逻辑在于，如果在操作过程中，版本号只比原来大1 ，那么就意味着操作过程中没有人对他进行过修改，他的操作就是安全的，如果不大1，则数据被修改过，当然乐观锁还有一些变种的处理方式比如cas

乐观锁的典型代表：就是cas，利用cas进行无锁化机制加锁，var5 是操作前读取的内存值，while中的var1+var2 是预估值，如果预估值 == 内存值，则代表中间没有被人修改过，此时就将新值去替换 内存值。

在上面demo中，通过在数据库中加入version自动，每次开启一个事务，当减完之后，验证版本是否符合预期，如果不符合就要进行事务的取消，执行创建订单失败

- lua + redis锁

通过lua脚本在redis之中进行原子性的操作，并在redis中进行订单的逻辑判断，最后把订单通过stream进行消息队列的分发，实现数据库的持久化。



#### lua脚本特性

原子性，这个是大家对lua脚本在redis中的第一印象，保证原子性的主要原因还是Redis采用了单线程执行模型。也就是说，当Redis执行Lua脚本时，Redis会把Lua脚本作为一个整体并把它当作一个任务加入到一个队列中，然后单线程按照队列的顺序依次执行这些任务，在执行过程中Lua脚本是不会被其他命令或请求打断，因此可以保证每个任务的执行都是原子性的。

#### 如何用go去加载lua脚本呢？

可以使用go-redis去执行`.lua`脚本

通过script.Load得到对应的lua脚本

```go
// createScript 通过加载[]byte{}加载lua脚本
script := createScript()
sha, err := script.Load(ctx, client).Result()
```



### Zap及ELK日志分析实践

系统通过Zap生成日志，推送到`Message Queue`中，之后再配置`logstash`将其写入到`ES`中，最后在`Kibana`中展示流程图如下

![ELK流程图](https://s2.loli.net/2023/10/14/baCn8HQ4o6ZEweu.png)

#### 定制化Zap实现

首先创建 logger，其有一个输出到 `console` 的 `zapcore`。和一个添加同步日志到 `redis` 的 `zapcore`，通过 `zapcore.AddSync` 可以将一个实现 `io.Writer` 接口的对象转为 `zap` 需要的 `WriteSyncer`。

`util/logger/logger.go`

```go
package logger

import (
	"gin-mall/elk_demo/util/rwriter"
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func NewLogger(rw *rwriter.RedisWriter) *zap.Logger {

	// 设置日志级别
	lowPriority := zap.LevelEnablerFunc(func(l zapcore.Level) bool {
		return l >= zapcore.DebugLevel
	})

	// 使用json格式日志
	jsonEnc := zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
	stdCore := zapcore.NewCore(jsonEnc, zapcore.Lock(os.Stdout), lowPriority)

	// rw实现io.Writer的接口
	syncer := zapcore.AddSync(rw)
	redisCore := zapcore.NewCore(jsonEnc, syncer, lowPriority)

	// 集成多个内核
	core := zapcore.NewTee(stdCore, redisCore)

	// Logger 输出console且标识调用代码行
	return zap.New(core).WithOptions(zap.AddCaller())
}
```

以及一个`redisWriter`

`rwriter/rediswriter.go`

```go
package rwriter

import (
	"context"
	"sync"

	"github.com/redis/go-redis/v9"
)

var rWriter *RedisWriter
var once sync.Once

type RedisWriter struct {
	cli     *redis.Client
	listKey string
	c       context.Context
}

func (w *RedisWriter) Write(p []byte) (int, error) {
	n, err := w.cli.RPush(w.c, w.listKey, p).Result()
	return int(n), err
}

// NewRedisWriter
func NewRedisWriter() *RedisWriter {
	once.Do(func() {
		rWriter = &RedisWriter{
			cli: redis.NewClient(&redis.Options{
				Addr:     "localhost:6379", // Redis 服务器地址
				Password: "",               // 如果有密码，设置密码
				DB:       0,                // 默认数据库
				PoolSize: 1000,
			}),
			listKey: "log_queue",
			c:       context.Background(),
		}
	})
	return rWriter
}
```

通过运行Test函数，可以看一下Redis中是否更新

#### 搭建ELK环境