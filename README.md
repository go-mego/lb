# Load Balancer [![GoDoc](https://godoc.org/github.com/go-mego/lb?status.svg)](https://godoc.org/github.com/go-mego/lb)

透過 Load Balancer 套件能夠以負載平衡的方式來平均連線請求。

# 索引

* [安裝方式](#安裝方式)
* [使用方式](#使用方式)
    * [取得節點](#取得節點)
    * [查詢模式](#查詢模式)
* [服務探索連動](#服務探索連動)
    * [Consul](#Consul)

# 安裝方式

打開終端機並且透過 `go get` 安裝此套件即可。

```bash
$ go get github.com/go-mego/lb
```

# 使用方式

此套件有許多使用方式與支援不同的服務探索中心，下列是目前支援的列表。

* 簡易本地列表
* Consul
* Etcd（_尚未支援_）
* Eureka（_尚未支援_）
* ZooKeeper（_尚未支援_）

`lb.New` 能夠建立一個最基本的負載平衡中介軟體，建立時可以傳入 `&lb.Nodes` 來指定固定的節點位置供稍後負載平衡配發使用。建立後傳入 Mego 引擎的 `Use` 即可作為全域中介軟體在所有路由中使用相同的負載平衡器。

基本的負載平衡器僅能使用固定的列表且不能移除伺服器。若需要有更彈性的使用方式，請參考下列與服務探索中心搭配的用法。

```go
package main

import (
	"github.com/go-mego/lb"
	"github.com/go-mego/mego"
)

func main() {
	m := mego.New()
	// 建立一個負載平衡中介軟體，指定相關節點位置並作為全域中介軟體使用。
	m.Use(lb.New(&lb.Nodes{
		"localhost:1234",
		"localhost:4567",
		"localhost:8910",
	}))
	m.Run()
}
```

負載平衡中介軟體也可以獨立用在不同的路由上。

```go
func main() {
	m := mego.New()
	// 將負載平衡中介軟體用於單一路由中。
	m.GET("/", lb.New(&lb.Nodes{
		// ...
	}), func(l *lb.Balancer) {
		// ...
	})
	m.Run()
}
```

## 取得節點

透過 `Get` 可以取得一個節點，而節點的 `String` 方法則可以將節點的資訊轉換成 IP 位置字串（含埠口）。

```go
func main() {
	m := mego.New()
	m.GET("/", lb.New(&lb.Nodes{
		"localhost:1234",
		"localhost:4567",
		"localhost:8910",
	}), func(l *lb.Balancer) {
		// 透過 `.Get()` 來透過負載平衡取得其中一個可用的伺服器。
		// 基本負載平衡會以輪詢當作查詢模式，每次會取得不同的伺服器，不會與上一個重複。
		fmt.Println(l.Get().String()) // 結果：localhost:1234
		fmt.Println(l.Get().String()) // 結果：localhost:4567
		fmt.Println(l.Get().String()) // 結果：localhost:8910
		fmt.Println(l.Get().String()) // 結果：localhost:1234
	})
	m.Run()
}
```

## 查詢模式

若要與其他服務探索中心搭配使用，這裡有幾種不同的查詢用法。雖然查詢方法不同，但是設置都是一樣的，詳細的設置請參考下方範例。

```go
func main() {
	m := mego.New()
	// 輪詢模式，依照順序呼叫不同伺服器。
	m.GET("/", lb.NewRoundRobin(&lb.Options{
		//...
	}), func(l *lb.Balancer) {})
	// 隨機模式，依照亂數呼叫不同伺服器。（*）
	m.GET("/", lb.NewRandom(&lb.Options{
		//...
	}), func(l *lb.Balancer) {})
	// （尚未支援）最近模式，永遠呼叫最低 Ping 值的伺服器。（*）
	m.GET("/", lb.NewClosest(&lb.Options{
		//...
	}), func(l *lb.Balancer) {})
	// （尚未支援）寬鬆模式，永遠呼叫最少連線數的伺服器。（*）
	m.GET("/", lb.NewFastest(&lb.Options{
		//...
	}), func(l *lb.Balancer) {})
	m.Run()
}
```

`（*）`：表示有可能與上一次的伺服器重複。

# 服務探索連動

負載平衡器可以與其他的服務探索中心相互連動，只要符合相關的 Interface 抽象介面也可以自己實作與其他探索中心的銜接功能。

## Consul

負載平衡也能夠使用 Consul 作為基礎，這能夠讓你有更大的彈性來取得伺服器位置，而不是僅能使用固定寫死的列表。

```go
package main

import (
	"github.com/go-mego/lb"
	"github.com/go-mego/sd/consul"
	"github.com/go-mego/version"
	"github.com/hashicorp/consul/api"
)

func main() {
	m := mego.New()
	// 以預設設定檔建立 Consul 客戶端。
	c, _ := consul.NewClient(api.DefaultConfig())
	// 以 Consul 為基礎，初始化一個輪詢負載平衡器。
	m.GET("/", lb.NewRoundRobin(&lb.Options{
		// 請參閱 Version 套件，這會產生一個 `1.0.0+stable` 的字串。
		Tag:    version.Define(1, 0, 0, "stable").String(),
		Name:   "Database",
		Client: c,
	}), func(l *lb.Balancer) {
		// 透過 `Get` 取得 Consul 服務探索中心裡其中一個
		// 帶有 `1.0.0+stable` 標籤的 `Database` 伺服器 IP 位置。
		l.Get().String()
	})
	m.Run()
}
```