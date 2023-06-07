# kong admin api 使用 go-kong 调用

## golang示例代码

```go
package main

import (
	"context"
	"encoding/base64"
	"fmt"

	"github.com/kong/go-kong/kong"
	"net/http"
)

// base64编码
func b64enc(s string) string {
	return base64.StdEncoding.EncodeToString([]byte(s))
}

func main() {
	// 调用 go-kong 插件
	// 创建Basic认证凭证
	username := "kong"
	password := "123456"
	auth := username + ":" + password
	basicAuth := "Basic " + b64enc(auth)

	c := kong.HTTPClientWithHeaders(nil, http.Header{
		"Authorization": []string{basicAuth},
	})

	// 使用kong.NewClient()创建一个 kong.Client 对象
	kongClient, err := kong.NewClient(kong.String("https://kong.example.com/api"), c)
	if err != nil {
		panic(err)
	}

	var defaultCtx = context.Background()

  // 查询 routes 列表的tag 条件
	listOpt := kong.ListOpt{
		Tags: []*string{
			kong.String("k8s-name:test-ipfs-gateway-kong"),
			kong.String("k8s-namespace:test-ipfs-cluster"),
		},
	}
  // 查询routes
	routes, _, err := kongClient.Routes.List(defaultCtx, &listOpt)

	if err != nil {
		panic(err)
	}
	//fmt.Println(listOpt)
	for _, route := range routes {
		fmt.Println(*route.ID, *route.Name)
	}
}

```

## 参考

https://github.com/Kong/go-kong

https://docs.konghq.com/gateway/latest/admin-api/