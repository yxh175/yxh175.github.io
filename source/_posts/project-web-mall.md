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