# golang计算文件ipfs cid

## 示例代码

```go
package main

import (
	"context"
	"fmt"
	"github.com/ipfs/boxo/coreiface/options"
	"github.com/ipfs/go-libipfs/files"
	"github.com/ipfs/kubo/core"
	"github.com/ipfs/kubo/core/coreapi"
	"github.com/ipfs/kubo/core/node"
	"os"
)

// 生成cid, 支持大文件
func cidTest() {
	// Create a new IPFS node
	ipfsNode, err := core.NewNode(context.Background(), &node.BuildCfg{Online: false})
	if err != nil {
		panic(err)
	}

	// Get the core API
	api, err := coreapi.NewCoreAPI(ipfsNode)
	if err != nil {
		panic(err)
	}

	// Read the file into memory
	file, err := os.Open("/Users/liujinye/Downloads/kubo/listen1_2.21.7_mac_x64.dmg")
	if err != nil {
		panic(err)
	}

	f := files.NewReaderFile(file)
	// Add the file to IPFS

	cid, err := api.Unixfs().Add(context.Background(), f, options.Unixfs.CidVersion(1))
	if err != nil {
		panic(err)
	}

	fmt.Println(cid.Root().String())
}

func main() {
	cidTest()
}
```

