# 新增 retry 能力

## 问题描述

etcd 注册成功之后，断开不会收到提示，也不会自动重连，导致实例一直在线但不可用。

## 解决方案

服务注册到 etcd 后，开启一个协程异步的按给定时间间隔 (ObserveDelay) Get etcd 中服务的状态。若存在问题，则进行
重试操作 (重新将服务 put 上去)，第一次重试失败就根据设置的重试次数 (MaxAttemptTimes) 和重试时间间隔 (RetryDelay) 继续
重试操作，重试次数用完还没重连成功就打印 error 日志，不在继续重试；若不存在问题，就按给定时间间隔 (ObserveDelay) 检查服务的状态。


## 配置

| 配置名称            | 默认值           | 介绍                    |
| :------------------ |:--------------|:----------------------|
| WithMaxAttemptTimes | 5          | 用于设置最大尝试次数，若为 0 即无限尝试 |
| WithObserveDelay       | 30 * time.Second | 用于设置正常情况下检查服务状态的延迟时间  |
| WithRetryDelay        | 10 * time.Second | 用于设置状态异常后的重试延迟时间      |

## 示例

若不需要自定义重试配置，使用 `etcd.NewEtcdRegistry()` 即可；

若需要自定义重试配置，使用如下代码：

```go
package main

import (
	"context"
	"github.com/kitex-contrib/registry-etcd/retry"
	"log"
	"time"

	"github.com/cloudwego/kitex-examples/hello/kitex_gen/api"
	"github.com/cloudwego/kitex-examples/hello/kitex_gen/api/hello"
	"github.com/cloudwego/kitex/pkg/rpcinfo"
	"github.com/cloudwego/kitex/server"
	etcd "github.com/kitex-contrib/registry-etcd"
)

type HelloImpl struct{}

func (h *HelloImpl) Echo(ctx context.Context, req *api.Request) (resp *api.Response, err error) {
	resp = &api.Response{
		Message: req.Message,
	}
	return
}

func main() {
	retryConfig := retry.NewRetryConfig(
		retry.WithMaxAttemptTimes(10),
		retry.WithObserveDelay(20*time.Second),
		retry.WithRetryDelay(5*time.Second),
	)
	r, err := etcd.NewEtcdRegistryWithRetry([]string{"127.0.0.1:2379"}, retryConfig)
	if err != nil {
		log.Fatal(err)
	}
	server := hello.NewServer(new(HelloImpl), server.WithRegistry(r), server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{
		ServiceName: "Hello",
	}))
	err = server.Run()
	if err != nil {
		log.Fatal(err)
	}
}
```




