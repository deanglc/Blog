---
title: "go-redis.md"
description: "> https://github.com/golang/go/blob/master/src/runtime/HACKING.md 待翻译"
tags: [ "golang", ]
categories: [ "golang", ]
keywords: [ "golang", "goroutine" ]
isCJKLanguage: true

date: 2020-06-21T11:00:53+08:00
draft: true
---


[https://gopher.cc/go-redis-%E8%BF%9E%E6%8E%A5%E6%B1%A0-529.html](https://gopher.cc/go-redis-连接池-529.html)

```go
// 定义redis链接池
var client *redis.Client

// 初始化redis链接池
func init() {
  db, err := beego.AppConfig.Int("redisDB")
  if err != nil {
    logs.Error("redis-db", err)
  }
  client = redis.NewClient(&redis.Options{
    Addr:         beego.AppConfig.String("redisAddr"),     // Redis地址
    Password:     beego.AppConfig.String("redisPassword"), // Redis账号
    DB:           db,                                      // Redis库
    PoolSize:     40,                                      // Redis连接池大小
    MaxRetries:   3,                                       // 最大重试次数
    IdleTimeout:  5 * time.Second,                         // 空闲链接超时时间
    MinIdleConns: 5,                                       // 空闲连接数量
  })
  pong, err := client.Ping().Result()
  if err == redis.Nil {
    logs.Info("Redis异常")
  } else if err != nil {
    logs.Info("失败:", err)
  } else {
    logs.Info(pong)
  }
}

type Redis struct{}

func (r Redis) Get(key string) (string, error) {
  result, err := client.Get(key).Result()
  if err != nil {
    return "", err
  }
  return result, nil
}
```

![Options相关说明](assets/1.jpg)

---

https://www.tizi365.com/archives/292.html