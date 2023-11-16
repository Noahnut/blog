---
title: Mage 教學 基礎篇
tags: golang
date: 2023-11-16 13:28:42
---



## 前言

[Mage](https://github.com/magefile/mage) 是一套將 Makefile 用 Golang 取代掉讓工程師免於熟悉其他語言的困惱的 Package，另外也利用程式語言的靈活性達到比 Makefile 可以做更多事情，這篇文章主要是教學 Mage 怎麼在專案中使用，安裝的部分可以去 [Mage Github](https://github.com/magefile/mage) 上看，會有更新更即時的安裝資訊。


## Mage Go 檔案的基礎格式

Mage 在執行 `mage` 這個指令時都會去找哪些檔案有 `go:build mage` tag，有這些 `tag` 檔案內部的 **Public** 函式名稱都會被當成要執行的 `target`，以下面這個簡單的程式為例


```go
// go:build mage

package main

import "fmt"

func Build() {
	fmt.Println("I'm build")
}

func Hi() {
	fmt.Println("hi")
}
```

從上面的程式可以看到註解是有 `go:build mage` 的 tag，並且有 `Build` 跟 `Hi` 兩個 Public 函式。

來看看在終端機上執行 `mage` 指令會發生什麼事。
執行 `mage` 可以顯示出目前可以執行的指令。

```shell
$ mage
Targets:
  build
  hi
```
執行 `mage build` 可以發現 `target`  的 build 被執行，而被執行的內容就是 `Build` 函式並產出我們預期的內容。
```shell
$ mage build
I'm build
```
基本上從這邊看起來 `mage` 就像是命令列的產出，方便我們去執行我們所定義好的內容。


### Mage 基礎 Golang 檔案建製

這邊會搭配簡單的 Golang 程式架構來介紹，如何用 `mage` 來取代我們通常用命令列或 Makefile 執行的 `build`、`run`、`test` 等等功能。

下面的程式架構是包含 main 與 test 的簡單 Golang 框架，接下來看看如何用 `mage` 來達到 `build`、`run`、`test` 的效果
```shell
./
├── cmd
│   ├── magecmd.go
│   └── magecmd_test.go
├── go.mod
├── go.sum
├── mage.go
└── main.go
1 directory, 6 files
```

下方的程式實作 `build`、`test`、`run` 的操作，接下來只需要在終端上分別執行 `mega build`、`mega test` 等等就可以呼叫對應函式內的功能。

```go
//go:build mage

package main

import "github.com/magefile/mage/sh"

func Build() error { // 編譯 golang 二進制檔案
	if err := sh.RunV("go", "mod", "download"); err != nil {
		return err
	}
	return sh.RunV("go", "build", "-o", "mage-test", "main.go")
}

func Test() error { // 執行單元測試
	return sh.RunV("go", "test", "-v", "./cmd/")
}

func Run() error { // Local run
	return sh.RunV("go", "run", "main.go")
}

func Remove() error { // 刪除編譯產出二進制檔案
	return sh.RunV("rm", "mage-test")
}
```

例如執行 `mega test` 就可以去執行指定的 go test。
```shell
$ mage test                                                                                                                                                                                   11:26:29  ✔
=== RUN   Test_MageCmd
--- PASS: Test_MageCmd (0.00s)
=== RUN   Test_MageCmdTwo
--- PASS: Test_MageCmdTwo (0.00s)
PASS
ok  	mage-test/cmd	(cached)
```


`mage` 也提供非常友善的終端指令執行函式，有客製化需求的朋友可以看一下裡面的內容，我這邊就用最簡單的 `RunV` 代表執行對應的指令並且都會將過程印到 Standard output。
這樣在 `mage.go` 中實作這些方法後，在終端命令列直接執行 `mage` 與對應的 `target` 就能執行裡面的動作。

## 結論

目前 `mage` 寫起來相對於 Makefile 對於 Golang 友善許多，再加上 Makefile 會有縮排的問題但是 `mage` 並不會有這個問題，目前 `mage` 簡單用起來是相當順手。
接下來會介紹 `mage` 本身怎麼去搭配 golang 架構去做整理跟 `mage` 有什麼樣的進階指令讓使用者使用起來更加直觀。
