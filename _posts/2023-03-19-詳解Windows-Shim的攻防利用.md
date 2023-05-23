---
layout: post
title: 詳解Windows Shim的攻防利用
subheading: A brand new starting point of my life.
author: Jeffrey
categories: jekyll
banner:
  video: 
  loop: true
  volume: 
  start_at: 
  image:
  opacity:
  background:
  height: 
  min_height: 
  heading_style: 
  subheading_style: 
tags: jekyll theme yat
sidebar: []
---

# 詳解Windows Shim的攻防利用

## 前言
---
Shim的字面意義為工業上所使用的墊片，主要是讓填埔兩個接合面間的空隙使其更加穩固。
而在程式設計領域，Shim則是一組提供應用程式相容性的小型函式庫，主要是透過攔截API呼叫或函式間傳遞的參數，使其重導向到指定的位址。簡單來說，就是程式掛勾 (hooking)的手法。
然而水能載舟亦能或行為攻擊者也可以利用Shim誘導程式執行惡意內容，來達到攻擊的目的。因此，本文將介紹Windows作業系統上Shim的攻擊和防禦方法。
Windows Shim Database預設路徑為: \Windows\AppPatch\sysmain

## 架構圖
---

## 攻擊面
---
攻擊者可以使用Shim來執行惡意應用程式，或者將惡意應用程式偽裝成合法應用程式。
攻擊者可以創建一個Shim層來修改應用程式的行為，例如將應用程式的輸出導向攻擊者所控制的伺服器。
攻擊者還可以利用Windows Shim來避免防病毒軟體的檢測。

## 防禦面
---
為了防止Windows Shim的攻擊，可以執行以下幾個步驟：
1. 禁用不必要的Shim層：請檢查系統中使用的Shim層，並關閉不必要的Shim層。
2. 更新防病毒軟體：防病毒軟體可以檢測和阻止惡意Shim層的執行。
3. 更新應用程式：應用程式開發人員可以修補Shim層的漏洞，並改進應用程式的安全性。

## 總結
---
總結來說，Windows Shim是一個有用的軟體工具，但它也是把雙面刃。為了防止Windows Shim的攻擊，我們需要關注和維護系統的安全性，以及定期更新軟體和應用程式。

## 參考資料
---
1. <a href="https://www.itread01.com/articles/1474605162.html" target="_blank">https://www.itread01.com/articles/1474605162.html</a>
2. [https://blog.30cm.tw/2018/03/windows-shim.html](https://blog.30cm.tw/2018/03/windows-shim.html)
3. [https://learn.microsoft.com/en-us/windows/win32/devnotes/shim-database-types](https://learn.microsoft.com/en-us/windows/win32/devnotes/shim-database-types)
4. [http://redplait.blogspot.com/2011/04/shim-handlers.html](http://redplait.blogspot.com/2011/04/shim-handlers.html)
5. [https://en.wikipedia.org/wiki/Shim_(computing)](https://en.wikipedia.org/wiki/Shim_(computing))
6. [https://tzworks.com/prototype_page.php?proto_id=33](https://tzworks.com/prototype_page.php?proto_id=33)
