---
title: Android 14 Root 生命鬥士
date: 2024-09-16 18:18:34
tags:
    - android
---

因為我的手機是一台古董 Pixel 5
不能用最新最潮的 KernelSU
再加上遇到幾個 module 過舊或是更新導致一瞬間原本能動的銀行 APP 全都失效
因此花了點時間將 root 環境重新隱藏起來

所以這篇就是一個順手記錄與各大金融 APP 拼命的結論
然後再順便從頭教學一下怎麼 root 自己的手機 XD

<!-- more -->

其實前面說我的系統 kernel 過舊不能用 KernelSU 也是不太對
只是我一直 build 不出可以正常開機的 kernel
以及 KernelSU 也宣佈從 `v0.9.5` 後不再支援通用內核以外的舊版或客製化內核
因此權衡一下還是繼續使用 Magisk 好了 XD

之前我繞 root 的環境挺單純的
就 Kitsune Magisk 26.4 + PlayIntegrityFix 而已
不得不說 Kitsune 原本就蠻強了
混淆了 package name 後
確實就已經讓許多金融 APP 抓不到了

## 起因

但幾週前
我更新了 Android 中的 Google Play 系統更新 (不是 Google Play 商店的更新)
導致 Zygisk 跟 LSPosed 19.2 會相衝
而 LSPosed 19.2 也是一個很久沒更新的東西了

以及我的 LINE Pay 突然被查殺
推測可能是 LINE 偷掃了 app list

這時候其實我還沒用 Hide My Applist
我的 LSPosed 是去做別的事情用的
但現在 LSPosed 也葛屁了

接著又加上 Kitsune 的 release 太久沒更新
還有之前用了 Kitsune beta 有爆炸的經驗
剛好又看到 Magisk 有最新的 beta 可以用
於是就想說來個大換血好了 XD

總之完全是種種因素疊加
導致我除了想大換血以外
也想更結構性的去理解每個元件之間的搭配

那接著就直接開始動手操作的環節啦 ~

## 前置作業

在安裝前
我們要先準備兩樣東西

1. adb 環境
2. 你手機系統的 boot.img
3. 解鎖 bootloader

- 第一點需要手動配置一下系統環境

> 如果你不想要安裝又肥又大的 Android Studio 的話

首先到 [這裏](https://developer.android.com/tools/releases/platform-tools) 來下載 `platform-tools`
然後配置進你的環境變數會比較方便
當然你也可以直接進資料夾做使用

- 第二點
各個廠商自己提供的方式都不一樣
我就以 Pixel 5 來舉例
Google 會將 Pixel 系列的系統映像檔或是 OTA 升級包統一都放在 https://developers.google.com/android/images 當中

然後我們需要檢查一下手機目前的系統版本
可以至 設定 -> 關於手機 -> Android 版本 -> 版本號碼 中確認

![image](1.png)

以我的為例
我的版本號碼為 `UP1A.231105.001.B2`

因此找到 Pixel 5 欄位的相同版本

![image](2.png)

然後點右邊的下載即可拿到官方映像檔

下載完成後要解壓縮
再將裡面的 `image-xxxx.zip` 解壓縮
會得到 `boot.img`

![image](3.png)

要將這個先傳進手機裡

- 第三點

這個根據每家廠商不同
可能要再請讀者自行上網找資料了
只是這步驟也是最重要的
因為沒有解鎖的話
是不能刷任何其他韌體進去的

## 安裝

這邊內容除了安裝過程外
還會一邊穿插過程中遇到的套件間的問題
因此整個過程會有點長 XD


### 安裝 Magisk

首先當然就是要有請 root 界中的經典面具上場啦
我們選用最新的 beta 版本 [27007](https://github.com/topjohnwu/Magisk/releases/tag/canary-27007)
下載 github release 內的 `app-release.apk` 安裝

接著就按照 [官方文件](https://topjohnwu.github.io/Magisk/install.html) 找尋適合你手機的方式將 magisk 安裝好
因為每家廠商的手機對於 root 該刷的位置都有不同是適應方式
這邊我舊粗淺講一下我的步驟 (應該也是所有 Pixel 系列的步驟)

```bash
adb reboot fastboot
fastboot flash boot magisk-xxxx.img
fastboot reboot
```

刷好新的 boot.img 後
重啟進入 magisk 應該就會看到上面那區的 magisk 寫了已安裝的版本而不是 `N/A` 了

![image](4.png)

### 安裝一堆模組

確定 magisk 成功安裝後
下一步就是要把重要的模組一一安裝進去
總共會需要以下幾個 magisk 模組

- [PlayIntegrityFix](https://github.com/chiteroman/PlayIntegrityFix/releases/tag/v17.5) (通常簡稱 PIF)
- [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext/releases/tag/v1.1.0)
- [Shamiko](https://github.com/LSPosed/LSPosed.github.io/releases/tag/shamiko-357)
- [LSPosed MOD](https://github.com/mywalkb/LSPosed_mod/releases/tag/v1.9.3_mod)

這邊要注意絕對不要啟用 magisk 設定中自帶的 zygisk
不確定是不是原本自帶的有問題導致 root 會一直被抓到
直接使用 zygisk-next 來替換掉才可以正常隱藏

另外就是不要使用原本的 LSPosed 1.9.2
因為那個版本已經非常久沒更新
會導致在最新的 Google Play 系統更新 202408 的版本會出現問題
導致 Zygisk 會無法啟動
使用這個 MOD 版可以修正此問題

至於 PIF 就是為了能過 safetynet 檢查
讓它至少是 BASIC 狀態來規避一些檢查

shamiko 則是基於 zygisk 的模組
可以隱藏 Zygisk, Zygisk 模組以及 Magisk root 的一個閉源模組
使用時不可以開啟 Magisk 的強制黑名單

安裝順序推薦 Zygisk-Next 優先
然後才安裝其他基於 Zygisk 的模組 (例如 Shamiko 以及 LSPosed)
而 PIF 要先裝或後裝都沒差

最後則是在 LSPosed 模組安裝完成後
添加一個叫做 [Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist/releases/tag/V3.2) 的應用程式 (通常簡稱 HMA)

### 設定 root 隱藏躲避檢查

在模組都安裝完畢後
要開始進行 root 隱藏

首先要藏起來的是 magisk app 本身
這個從 app 的設定中
有一個 "隱藏 Magisk" 的選項
選擇後可以自己輸入想要的 app name
我是為了怕找不到
所以用一點變形的字串 `Mag1sk` 來方便辨識
但又不要以原本的 magisk 來命名

轉換結束後
應該會發現 app 清單中多了一個初始 android 娃娃頭像的 app

![image](5.png)

而這個 app 的 package name 則是一組亂數產生的包名
因此可以繞過大多數對於 app list 進行黑名單比對的檢查方法

再來要進入 "設定黑名單" 當中
將你要用到的 APP 全部勾起來
來使 MagsikHide 生效

通常到這步驟
多數的銀行 APP 應該已經可以穩定通過了

### 規避 app list 掃描

最後要隱藏的部分則是某些 app 會掃描 app list 的行為
我們要透過 HMA 來 hook 這些掃描行為
進而達到掃不到特定跟 root 有關的 app

先點開上面通知欄中的 LSPosed 管理介面

![image](6.png)

然後在第二項的模組清單
將 HMA 啟用

![image](7.png)

接著回到 app 清單中找到它點開開始設定

![image](8.png)

再來點開模板管理
選擇新增一個黑名單模板
然後填一個你喜歡的名字

![image](9.png)

這邊有兩個選項分別為

- 對應用程式執行攔截
- 對應用程式啟用

這邊中文挺怪的
總之可以理解為

- 要把哪些 app 藏起來不被掃到
- 要讓哪個 app 看不到上面藏起來的 app (有點繞口 XD)

總之只要先調上面那個就好

這裡我們主要要藏兩個東西

1. HMA app 本身
2. Magisk 混淆後的 app

勾起來後
記得一直上一步退回 HMA 主畫面
不然有時候他會沒儲到

再來就是進入 "管理應用程式"
然後找到你要的 app (這邊我範例使用 LINE)

![image](10.png)

點選 "啟用隱藏"
然後記得在下面的模板設定使用前面新增好的模板
然後一樣一直上一頁來讓他觸發儲存
接著就可以重新開機
來確認一下是否能成功進入 LINE Pay 做使用囉

### 測試隱藏是否生效

如果前面的步驟都有做正確
基本上到這裡用到的技巧幾乎能躲過所有的金融 APP 檢測了

如果還是沒有成功
這裡推薦幾個 APP 可以讓你檢查自己的環境是否有設定正確
分別是

- [Momo](https://github.com/apkunpacker/MagiskDetection/blob/main/Momo-v4.4.1.apk) (原始版本存在某一個 tg 頻道裡)
    - 檢查系統 root 環境是否有藏好
    - 跟 magisk 有關的設定

- [Applist Detector](https://github.com/Dr-TSNG/ApplistDetector/releases)
    - 檢查你環境中有沒有什麼奇怪的 app 或是 api 跟 root 有關
    - 通常跟你的 HMA 有沒有藏好有關

- YASNAC (Play 商店找得到)
    - 檢查你的 safetynet 是否有 pass
    - 跟 PIF 有關

![image](11.png)

![image](12.png)

![image](13.png)


目前我手頭上有在使用的銀行跟行動支付工具按照上面的方式都是可以繞過檢查
至於更多的其他 APP 能不能繞過就不確定了
或是可能哪天一更新又全部被查殺也是有機會 XD
因為檢測手法千奇百怪
只能等真的遇到才能想辦法找出程式的判斷邏輯去應
