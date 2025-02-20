---
title: Web Security Academy
published: 2025-02-18
description: ''   
image: ''
tags: []
category: 'Port Swigger'
draft: false
lang: 'en'
---
# API Testing

## 24/29: Exploiting server-side parameter pollution in a query string


:::note
Bài này yêu cầu bạn sử dụng Burp Suite tìm lổ hổng bằng cách chèn các kí tự như `#, %,...` vào *URL* (nên test cả url encoding)
:::

Mình thử khá nhiều kí tự như %20, %23,.. thì đến ký tự %26 (&) thì server trả về 1 lỗi "Parameter is not supported"
Vốn dĩ kí tự này là để thực thi lệnh nếu lệnh trước thành công, vậy tức là mình có thể thêm 1 hoặc nhiều argument đằng sau 

=> Có thể Command injection 

Nhưng không biết cần bao nhiêu argument để payload được xử lý, mình quyết định truncate payload bằng kí tự # cho từng argument

1. Giữ nguyên argument đầu tiên (username), sau đó Send

Field not specified error, vậy là thiếu argument tên là `field`


2. Thêm `field` vào và Send

Invalid field error, boom, vậy chỉ cần 2 argument là `username` và `field`

## Viết payload

Cần phải viết payload vào trường username, ko được thêm trường khác vào. Mình sẽ để tạm parameter cho field là tmp

> %26 = &
>
> %23 = #

```
    username=administrator%26field=tmp%23p
```

Giờ đi tìm parameter cho trường field là oke

Tìm ở đâu? Coi burp suite thử, thấy ngay 1 file .js khá weird, bật lên đọc 1 hồi mình thấy 1 function forgotPassword() đúng cái api mình đang tấn công

> ?reset_token={resetToken}

Có vẻ browser sẽ check resetToken của client với resetToken trên server

=> Vậy vô server lấy resetToken của administrator là bypass.

```
    username=administrator%26field=reset_token%23
```

Có token rồi thì vào api resetToken với tham số là token đó

Sau đó tạo mật khẩu mới, đăng nhập administrator, vào panel rồi delete user carlos