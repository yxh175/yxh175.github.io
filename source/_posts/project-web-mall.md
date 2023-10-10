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

#### 权限拦截及双Token的实现

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