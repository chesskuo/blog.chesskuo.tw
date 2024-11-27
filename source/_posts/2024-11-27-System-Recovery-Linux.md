---
title: System Recovery - Linux
date: 2024-11-27 16:33:40
tags:
	- mgnt
---

剛好最近無聊在研究如果碰的到實體機器
但卻沒有帳密 或是忘記帳密時
有沒有什麼手段可以進去系統改掉密碼

起初是研究 windows 該怎麼辦到
後來想說連 linux 一起研究
於是就產生了這篇

<!-- more -->

## 直接進入主題

這次使用的測試環境如下
- Debian 12.8
- GNU GRUB 2.06 (其實就是系統裝起來預設自帶的)

首先開機進入 grub 畫面

![](1.png)

按照底下敘述
按 `e` 可以進入編輯模式

![](2.png)

接著滑到最下面有一條 `linux` 的欄位

![](3.png)

根據 [grub 官方文件](https://www.gnu.org/software/grub/manual/grub/html_node/linux.html#linux) 可以知道後面需要傳遞 linux kernel image 以及相關參數
詳細資訊可以從 [kernel.org 官方文件](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html) 查詢

我們目前只需要知道 `ro` 代表之後啟動的系統掛載起來是唯讀
因此需要改成 `rw` 才可以更動系統內的檔案

接著要多一個參數 `init` 在最尾部
這是控制啟動系統後要載入的程式
直接設定成 `/bin/bash` 就好

![](4.png)

最後按 `ctrl` + `x` 即可用這次變更的設定檔進入系統
成功後會長這樣

![](5.png)

這邊要注意
如果維持 `ro` 進入系統的話
會無法進行任何修改

![](6.png)

有正確掛載進可寫的系統應該要可以如下

![](7.png)

我們可以看到系統內有 `root` 跟 `user` 兩個使用者

![](8.png)

這邊就拿 `root` 忘記密碼的情境示範
嘗試修改 `root` 密碼為 `root2` 並且進入系統

![](9.png)

最後記得打個 `sync` 讓更動確定寫回 file system
因為在這個狀態下的系統非常陽春
沒有太多其他的底層維護 daemon 介入
因此需要盡量保證系統穩定

最後就可以直接重開機啦
但是會發現打了 `reboot` 卻噴 error

![](10.png)

原因是我們在最一開始 grub 啟動的時候
是直接用 `/bin/bash` 進入的
並沒有讓 systemd 啟動
因此沒辦法維護系統狀態
這邊就需要透過手工對 kernel 送重啟訊號了

首先要啟用 `sysrq` 的功能
可以參考 [kernel.org 官方文件](https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html)

`1` 代表啟用全部的功能
![](11.png)

```bash
echo 1 > /proc/sys/kernel/sysrq
```

然後送 `b` 給 `/proc/syseq-trigger` 代表立刻重啟系統
![](12.png)

```bash
echo b > /proc/sysrq-trigger
```

開機完成正常的系統後
輸入剛剛改過的密碼 `root:root2` 就成功進入系統了

![](13.png)
