---
title: System Recovery - Windows
date: 2024-11-26 16:44:29
tags:
	- mgnt
---

一切的起因就是某天晚上一個朋友說他的 windows 忘記密碼
然後錯太多次被鎖起來
然後他的電腦有啟用 bitlocker

本來想找公司 infra 幫忙解
但他說好像印象中以前 infra 是用 WinPE 解的樣子
結果 bitlocker 的 key 就變了

但我想想不太合理
改密碼是 bitlocker 解鎖硬碟後對檔案系統的行為
怎麼會改個密碼連 bitlocker key 都變了
於是就花了點時間研究到底 windows 忘記密碼的還原流程到底是如何

<!-- more -->

先說結論有兩種方式
1. 用原本那台忘記密碼的 windows 系統即可
2. 把硬碟掛去一台 linux 底下用工具救

測試環境
- Windows 11
- enabled BitLocker

## recovery from same windows system

![](1.png)

不囉嗦
總之登不進去就對了

![](2.png)

然後到右下角的電源鈕
按住 `shift` + 右下角的 Restart
就會進到 recovery mode

![](3.png)

接著點 Troubleshoot

![](4.png)

然後點 Advanced options

![](5.png)

這邊就會出現 Command Prompt 的選項了
點下去會先要你輸入 BitLocker 的解密金鑰
然後就會送你一個 `cmd.exe`

![](6.png)

![](7.png)

接著要找一下你的系統槽是在這個 recovery mode 的哪
可以用 `wmic` 列一下邏輯槽出來看一下

```cmd
wmic logicaldisk get deviceid,volumename
```

![](8.png)

通常沒意外都是 C 槽才對
就直接 `cd` 切換過去吧

![](9.png)

到這邊我們就要先了解一個事情
在鎖定畫面的狀態下
右下角第二個那個小人圖標本身其實是一個 exe (`Utilman.exe`)

![](10.png)

點開後的每一個功能本身也都是一個 exe

![](11.png)

所以思路就是去替換掉其中一個 exe 來啟動 `cmd.exe`
我們這邊就拿小人圖標開刀

直接切過去 `System32` 裡面
然後將 `Utilman.exe` 先備份一下
最後將 `cmd.exe` 複製成 `Utilman.exe`

![](12.png)

到這邊已經差不多了
可以關掉 cmd 然後正常進入系統

開機完成後
去點那個小人圖標就會跳出一個高權限的 `cmd.exe` 了

![](13.png)

最後直接舒舒服服的 `net user` 改你要的帳號的密碼即可
然後就可以舒舒服服去登入系統啦

![](14.png)

![](15.png)

---

## recovery from linux

這邊就有點小麻煩
因為要把硬碟拆下來插去另一台主機上
所以如果能用原本的 windows 系統主機來復原當然是最好 XDD

接著進入正題
把硬碟插去 linux 上
先用 `fdisk` 檢查一下到底哪一顆是我們要的

![](16.png)

可以看到有一個寫著 `Microsoft basic data` 就是我們的目標
而他在的分割區為 `/dev/sda3`

接著我們需要使用到一個工具叫做 `dislocker`

它的用處是可以處理 Windows BitLocker 硬碟解密後掛載成 FUSE 結構
然後再讓我們把裡面的 NTFS 系統 `mount` 起來用
這樣就可以達到真正修改加密磁碟後裡面的內容

安裝的話直接

```bash
apt install -y dislocker
```

使用上首先要創兩個資料夾準備給掛載用
這邊統一在 `/mnt/` 底下做

![](17.png)

如同上面所說
`dislocker` 需要先將 BitLocker 解密然後形成 FUSE 結構
所以需要有一個 `decrypt/` 資料夾來存放 block file `dislocker-file`

```bash
dislocker -V <你上面看到的 Windows 分割區> -p<bitlocker-key> -- <掛載點>
```

![](18.png)

> 在這個步驟如果出現錯誤可能是因為 apt 預設裝的 dislocker 版本有問題
> 改從 [官方 Github](https://github.com/Aorimn/dislocker) 拉下來自己編譯應該就可以解決
> 我這邊測試時用的是 Debian 就有撞到
> 如果改用 Kali 就沒有問題

接著將這個 block file 掛載起來在 `c/` 資料夾中即可看到 Windows 系統檔案了

```
mount -o loop /mnt/decrypt/dislocker-file /mnt/c
```

![](19.png)

> 這邊也有機會踢到一個問題就是找不到 NTFS driver
> 記得要 `apt install nfts-3g` 一下

掛載完成後
我們需要使用 `chntpw` 來對 Windows 的 SAM 進行操作以還原帳號密碼

```bash
apt install -y chntpw
```

接著去讀取一下 SAM 的內容看看有哪些使用者存在
SAM 在 Windows 系統中的路徑為 `C:\Windows\System32\config\SAM`

```bash
chntpw -l Windows/System32/config/SAM
```

![](20.png)

這邊就拿 `User` 舉例

```bash
chntpw -u User Windows/System32/config/SAM
```

![](21.png)

`chntpw` 只能直接清空密碼
直接選 `1` 然後退出即可

![](22.png)

接下來就按照順序 `umount` NTFS 磁碟跟 Bitlocker 區塊
然後就可以把硬碟插回去給 Windows 開機看看 `User` 這位使用者是否可以無密碼直接登入

```bash
umount /mnt/c
umount /mnt/decrypt
```

![](23.png)

成功
不用密碼就能直接登入了

