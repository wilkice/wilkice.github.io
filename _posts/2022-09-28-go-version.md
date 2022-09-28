---
title: Go 编译的时候带上版本信息
tags:
  - dev
---
在我们平时使用程序的时候，程序的版本其实是一个很重要的信息。在 Go 语言编译程序的时候，怎么把版本信息尽可能方便的传递下去呢？本文谈谈我的想法。
<!--more-->

## 最开始的想法
最开始的时候，我采用的是一种很笨的办法，那就是直接在代码里面hard code，比如
```go
package main

import (
	"flag"
	"fmt"
)

func main() {
	version := "v1.0.1"
	v := flag.Bool("v", false, "show version")
	flag.Parse()
	if *v {
		fmt.Println(version)
	}
}
```
这样会有一个问题，当要发布新版本的时候，我什么时候才能修改 `version` 的值。
- 如果先修改 `version`的值，然后提交 commit。如果跑 CI 的时候发现有问题，这个时候需要修改代码，然后还需要记得修改 `version`，不然用 `-v` 打印出的两个版本一样。这样会带来很大的心理负担，需要随时注意`version` 的值。
- 如果先不动 `version` 的值，等 CI 跑通之后，再单独用一个 commit 来修改 `version` 的值也行。但是这样感觉不太优雅，同样也会带来心理负担，生怕忘记修改值了。

## 参考其他开源软件
因为上面方法的心理负担，于是我开始查看比较出名的一些开源库，比如 hugo, cloudflared，看看他们是怎么处理这个问题的。

### [hugo](https://github.com/gohugoio/hugo)
我首先看了 hugo 的处理，发现跟我上面的第二点做法有类似之处，也是用了一个单独的 commit 来处理这个版本信息
![](https://img-1301200364.cos.ap-guangzhou.myqcloud.com/CleanShot 2022-09-28 at 22.18.39@2x.png)

### [cloudflared](https://github.com/cloudflare/cloudflared)
然后我看了 cloudflared 的处理，在它的 [Makefile](https://github.com/cloudflare/cloudflared/blob/master/Makefile) 中有这样的处理（我删减了跟version无关的部分）
```makefile
VERSION       := $(shell git describe --tags --always --match "[0-9][0-9][0-9][0-9].*.*")
DATE          := $(shell date -u '+%Y-%m-%d-%H%M UTC')
VERSION_FLAGS := -X "main.Version=$(VERSION)" -X "main.BuildTime=$(DATE)"
LDFLAGS := -ldflags='$(VERSION_FLAGS) $(LINK_FLAGS)'

# 这一句是我删减了的
go build $(LDFLAGS) 
```
这里用到了 ldflags 来修改 main 中 `Version` 的值，不了解 ldflags 的朋友可以看看 Digital Ocean 的这篇[文章](https://www.digitalocean.com/community/tutorials/using-ldflags-to-set-version-information-for-go-applications)。

这个处理就比较友好了。在我们代码提交并且通过了 CI 之后，我们就可以打上 git tag 发布了。在 makefile 编译的时候，会把当前 git 的 tag 信息传递给我们的程序，这样我们不用一直记着来修改这个版本信息，也不用单独用一个 commit 来配置版本了。


