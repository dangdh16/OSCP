# **Blue**
## 1. Recon
![](2021-09-26-04-33-26.png)
> Mình lại dùng nmap để scan

```
nmap -T4 -sC -sV -p- --min-rate=1000 -oN nmap.log 10.10.92.219 -Pn
```
Bình thường sử dụng -sC nmap sẽ mặc định chạy những script default vì trong tập lib của nmap nhiều script mang tính xâm nhập tấn công, nên ko set default.

Để scan được vulneribility ta dùng `--script vuln`
```
nmap -T4 --script vuln -sV -p- --min-rate=1000 -oN nmap.log 10.10.92.219 -Pn
```
![](2021-09-26-04-36-58.png)
> Kết quả là có lỗi RCE ms17-010

## 2. Gain Access
*Lưu ý là trước khi làm bước này cần phải biết Metasploit*