# mac os golang使用go-sqlite3项目

## 安装交叉编译C/C++依赖

```
# 编译win环境需要
brew install mingw-w64
# 编译linux/arm 环境需要  --with-i486（x86 32-bit），--with-aarch64（ARM 64-bit），--with-arm（ARM soft-float），--with-arm-hf （ARM hard-float 需要armv7后支持）
brew install FiloSottile/musl-cross/musl-cross --with-i486 --without-x86_64 --with-arm-hf --with-arm --with-aarch64
```

最后输出

```
==> /usr/local/opt/make/bin/gmake install TARGET=aarch64-linux-musl
==> /usr/local/opt/make/bin/gmake install TARGET=arm-linux-musleabihf
==> /usr/local/opt/make/bin/gmake install TARGET=arm-linux-musleabi
==> /usr/local/opt/make/bin/gmake install TARGET=i486-linux-musl
🍺  /usr/local/Cellar/musl-cross/0.9.9: 7,081 files, 799.0MB, built in 94 minutes 22 seconds
```

## 编译golang 带 go-sqlite3的项目



```
CGO_ENABLED=1 GOARCH=arm  GOOS=linux   CC=arm-linux-musleabihf-gcc CGO_LDFLAGS="-static" go build -a -v -installsuffix cgo -o bin/bfs-data-detection .
```

参考：https://saekiraku.github.io/article/18577/