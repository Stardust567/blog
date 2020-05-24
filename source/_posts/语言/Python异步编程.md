---
title: Python异步编程
comments: true
tags:
  - GAP
  - Python
  - asyncio
categories: Python
abbrlink: 92c
date: 2020-05-24 12:24:06
---

python由于GIL（全局锁）的存在，不能发挥多核的优势，这点一直饱受诟病。而在IO密集型的网络编程里，异步处理比同步处理能提升成百上千倍的效率，弥补了python性能方面的短板。asyncio在python3.4版本引入到标准库，python3.5又加入了async/await特性。<!-- More -->

