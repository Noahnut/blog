---
title: Mage 教學 進階篇 (1)
tags: golang
date: 2023-11-17 20:53:26
---


## 前言

上一篇 [基礎篇](https://noahnut.github.io/2023/11/16/mage-tutorial/) 介紹 `mage` 的基礎使用方法，這一篇會帶入更多的花招讓 `mage` 的使用靈活度更高。
基本上這邊的內容都是參考官方文件並搭配自己的實驗而來的。


## 帶參數的 Target

使用 Makefile 的時候常常遇到有些需要從命令列讀指定特殊參數，如果用 Makefile 來寫做法通常都有點不直觀，而且還要另外檢查輸入參數的數量是否合法，而 `mage` 利用函式的參數作為輸入的參數大大減低不直覺性與提高維護性，並且針對參數的類別，`mage` 還可以做到檢查輸入型別是否正確，以下是帶參數 target 的範例。

```golang

//go:build mage

package main

import (
	"fmt"

	"github.com/magefile/mage/sh"
)


// 支援 bool, int, string, time.Duration
func Build(testBool bool, testOne int, testString string, t time.Duration) error {
	fmt.Println(testBool, testOne, testString, t)
}
```

然後我們在命令列執行

```shell
$ mage build true 123 testnum 10ms # bool 值，0 或 1 作為參數也可以
true 123 testnum 10ms                                                                                                           
```

目前 `mage` 的參數支援 4 種類別，分別是 `bool`、`int`、`string`、`time.Duration`，如果參數有不合法的類別這個 target 會直接無法使用，另外 `int` 這個類別只支援 `int` 如果是使用 `int64` 或 `int32` 等等也是會被當成不合法的類別。

如果對應參數所帶的類別錯誤，例如要 `int` 卻給他 `string`，`mage` 也是會直接出現錯誤的，例如以下。

```shell
$ mage build 1 ttt test 100ns
can't convert argument "ttt" to int
```

如果參數帶不夠 `mage` 也會給你對應的錯誤

```shell
$ mage build 1 123 test
not enough arguments for target "Build", expected 4, got 3
```

## 註解的用途
`mage` 也提供 `target` 的說明，只要在 `target` 的函式上輸入註解，這個註解就會變成 `mage` 的 `target` 說明

```golang
// 我是 test 的說明
func Test() error {
	return sh.RunV("go", "test", "-v", "./cmd/")
}
```

```shell
$ mage
Targets:
  test      我是 test 的說明
```


## 多個 target 同時執行

`mage` 另外一個非常有趣的功能是 `mage` 可以在一個指令下呼叫多個 `target`，在 `mage` 中這個 target 的執行順序是從左邊執行到右邊。
下面的程式是有兩個 target。

```golang
// 我是 build 的說明
func Build() error {
	fmt.Println("build")
	if err := sh.RunV("go", "mod", "download"); err != nil {
		return err
	}
	return sh.RunV("go", "build", "-o", "mage-test", "main.go")
}

// 我是 test 的說明
func Test() error {
	return sh.RunV("go", "test", "-v", "./cmd/")
}
```

如果我們在命令列輸入 `mage build test`

```shell
$ mage build test
build
=== RUN   Test_MageCmd
--- PASS: Test_MageCmd (0.00s)
=== RUN   Test_MageCmdTwo
--- PASS: Test_MageCmdTwo (0.00s)
PASS
ok  	mage-test/cmd	(cached)
```
可以發現先執行的 `build` 在執行 `test` (如果要做到非同步的操作，golang 的 go routine 可以幫助到你)。

如果其中有 `target` 是有需要帶參數的，順序也是一樣的

```shell
$ mage build 123 t test
build 123 t
=== RUN   Test_MageCmd
--- PASS: Test_MageCmd (0.00s)
=== RUN   Test_MageCmdTwo
--- PASS: Test_MageCmdTwo (0.00s)
PASS
ok  	mage-test/cmd	(cached)
```


## 下期預告

下一章會介紹以下功能

1. Context 在 `mage` 中的用途
2. NameSpace
3. Zero Install Options