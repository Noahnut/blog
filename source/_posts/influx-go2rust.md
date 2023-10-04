---
title: InfluxDB 從 Golang 換成 Rust !
tags:
  - backend
  - database
cover: cover.png
date: 2023-10-04 14:49:00
---


## 故事
最近從 CHANGE LOG 的 Prodcast 聽到 influxDB 將整個 Database 改用 Rust 重寫，這已經是不知道聽到第幾個從其他語言換到Rust 的案例，雖然還沒有正式的文章跳出來說 influxDB 從 Golang 換成 Rust 後效能提升多少，但是 CTO 在 Reddit 有提到為什麼他們從 Golang 換成 Rust，而且早在 2020 年的時候，influxDB 就有發布這個文章 [Announcing InfluxDB IOx - The Future Core of InfluxDB Built with Rust and Arrow](https://www.influxdata.com/blog/announcing-influxdb-iox/)，這篇文章中就有提到他們目前在 influxDB 遇到什麼樣的問題，並開始考慮轉用 Rust 來解決他們遇到的問題。

## 為什麼要從 Golang 轉成 Rust ?
從文章跟 Reddit 的回覆可以了解，influxDB 在 2020 的時候就遇到一些效能上的問題，尤其是想要達成 `unlimited cardinality` 的功能時就會需要將底層架構改寫，所以 CTO 就決定反正都要改寫，我們不如使用更穩定、效能更好的語言來做新的架構，他們當時就選擇 Rust，而他會選擇 Rust 主要是因為以下理由
1. No garbage collector
2. Fearless concurrency
3. Performance
4. Error handling
5. Crates

的確 Rust 的效能與沒有 GC 但也不會有 Memory Leak 的功能，非常適合作為底層的 Database 的開發語言，不會有任何 GC 的效能損耗。

## 結論
InfluxDB 從 Golang 轉成 Rust 的理由滿有趣的，但是在實作這些 Application 的時候把語言特性考慮進去也是非常重要的一件事，Discord 從 Golang 轉成 Rust 後也可以看到明顯的效能提升。
