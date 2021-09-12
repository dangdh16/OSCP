# Firewall
## 1. Khái niệm firewall
Linux firewall là một thiết bị để kiểm tra network traffic (In/Out) và đưa ra quyết định chặn hay cho qua. Trong hệ điều hành Linux, tường lửa được thực thi tốt nhất bằng cách sử dụng netfilter. Đây là một kernel module quyết định packet nào được phép đi vào (input) hoặc đi ra bên ngoài (output).

## 2. Hoạt động của firewall
### 2.1 Firewall: stateless and stateful filtering mechanisms
**Packet filtering** : Các giao tiếp trong mạng thường chia nhỏ thành các phân đoạn thành packet nhỏ (MTU) 1500byte (thường thế). Và ở mỗi `header` có những thông tin để có thể xử lý. Packet filtering là cơ chế làm nhiệm vụ phân tích các `header` của các layer trong tcp/ip, dựa vào những rule mà admin đặt ra để có thể "cho qua" hoặc "chặn lại". *Kiểm tra gói tin dựa vào các thông tin: src des IP, src des port, protocol*
- *Stateless filtering*: Đánh giá các gói tin 1 cách độc lập. Có nghĩa là tất cả các gói tin qua mạng mới hay cũ đều phải áp dụng rule được định sẵn. Cơ chế này tạo ra quy tắc cho lưu lượng, tức là đầu vào thế này -> thì đầu ra phải thế này. Cơ chế áp dụng cho các kết nối với các mạng khác nhau, vì mạng này ko thể có dữ liệu về mạng khác từ trước được. => nhanh hơn *stateful filter*
- *Stateful filtering*: Được tạo ra để giải quyết các vấn đề bảo mật (ex: spoof MAC,..). Cơ chế filter này dựa trên một cơ sở để filter gọi là : `connection table` or `status table`. Với `status table` mỗi connection đều được register. Khi 1 packet gửi đến trước khi áp rule, `stateful firewall` sẽ check trong `status table` xem là đã có kết nối nào hay chưa, nếu đã có và hợp lệ thì cho qua, ko cần áp rule lên, nếu ngược lại thì chặn.

**Netfilter**: là một framework của kernel cho phép các ứng dụng tương tác với packet. Netfilter hooks:
![](img/2021-09-04-23-50-10.png)
> Những hooks này thực thi trực tiếp với linux kernel
- `NF_IP_PRE_ROUTING`: Đưa ra cách quyết định xử lý với một packet ngay khi tới network interface. Ở đây có thể drop packet, NAT, hoặc đơn giản là để nó tự đi qua.
- `NF_IP_LOCAL_IN`: Chứa các rule để kiểm soát truy cập từ internet, ví dụ như muốn chặn hay mở port từ IP khác đến. 
- `NF_IP_FORWARD`: Chịu trách nhiệm forward các packet như là bộ định tuyến.
- `NF_IP_LOCAL_OUT`: Chịu trách nhiệm cho quá trình truy cập ra bên ngoài, ví dụ như để quản lý các packet gửi ra bên ngoài, bạn ko thể gửi gói tin ra ngoài mà chưa được phép ấy.
- `NF_IP_POST_ROUTING`: Đây là nơi mà packet đi ra khỏi máy tính của mình. 

Các trường hợp packet sẽ đi và qua hooks (Chain Traversal Order) như sau:
- `1-2`: Gói tin được gửi đến máy của bạn, và ở LOCAL_IN sẽ có rule để có chấp nhận `packet` hay không
- `1-4-5`: Máy của bạn được coi như là bộ định tuyến (router)
- `3-5`: Máy của bạn gửi gói tin ra bên ngoài.

## 3. Các loại firewall
- Firewalld, ufw đều là những công cụ giao diện để điều khiển iptables, tuy nhiên nó không đầy đủ chức năng (Chức có những chức năng NAT, log,..). Nên ta tìm hiểu về iptables.
- Iptables thực chât là giao diện để admin tương tác với netfilter. Hiểu là iptables là frontend và netfilter là backend.

## 4. Cấu trúc firewall
### 4.1 Cấu trúc Iptables:
- **chain**: Có 5 chain trong iptable, Mỗi chain tương ứng với netfilter hooks chịu trách nhiệm với packet: `PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT` và `POSTROUTING`.
- **table**: Các table có nhiệm vụ khác nhau :  `filter`, `nat`, `mangle`, `raw` và `security`. table nat (NAT) chịu trách nhiệm chuyển đổi địa chỉ mạng.
- **target**: Chỉ định trạng thái của packet: `ACCEPT`, `DROP`, `RETURN`, `DNAT`, `LOG`, `MASQUERADE`, `REJECT`, `SNAT`, `TRACE` và `TTL`. 
### 4.1 Chain iptables
Nó giống như `netfilter hooks` thôi. khác cái tên thôi, iptables thực hiện trên user space gọi xuống netfilter thực hiện trong linux kernel.
### 4.2 Table iptables
Trong mỗi `table` sẽ có những `chain` nhất định được sử dụng. Bảng table được mặc định nếu ko chỉ định. Vì là bảng dùng nhiều nhất.
| Tables↓/Chains→ | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
|-----------------|------------|-------|---------|--------|-------------|
| filter | | ✓ | ✓ | ✓ | |
| nat (DNAT) | ✓ | | | ✓ | | |
| nat (SNAT) | | ✓ | | | ✓ |
| mangle |  ✓ | ✓ | ✓ | ✓ | ✓ |
| raw | ✓ |  |   | ✓ | 
| security | | ✓ | ✓ | ✓ | |

- `filter`:Trong table này, bạn sẽ quyết định xem packet có được phép vào (input) hoặc ra (output) khỏi máy tính của bạn hay không
- `nat`: nat gồm 2 loại: DNAT : Thay đổi địa chỉ đến của của gói dữ liệu khi cần thiết, SNAT: Thay đổi địa chỉ nguồn của gói dữ liệu khi cần thiết.
- `mangle`: Thay đổi các bit dịch vụ trong IP header: TTL (Time to live), TOS (Type of service)...
- `raw`: xử lý packet raw chưa qua xử lý. Chủ yếu là để theo dõi trạng thái kết nối.
- `security`: là bảng được sử dụng cho Mandatory Access Control (MAC) - kiểm soát truy cập bắt buộc đối với các rule về network. Có nhiệm vụ đánh dấu những gói tin có liên quan đến context của SELinux, xử lý gói tin.

### 4.3 Target iptables
Hành động áp dụng cho các gói tin được gọi là `target`.
- ACCEPT: chấp nhận gói tin, cho phép gói tin đi qua hay đi vào hệ thống.

- DROP: loại bỏ gói tin, không phản hồi lại gói tin giống như việc gói tin đó được gửi đến một hệ thống không tồn tại.

- RETURN: Dừng thực thi xử áp dụng rules tiếp theo trong chain hiện tại đối với gói tin. Việc kiểm soát sẽ được trả về đối với chain đang gọi.

- REJECT: Thực hiện loại bỏ gói tin và gửi lại gói tin phản hồi thông báo lỗi. Ví dụ: 1 bản tin “connection reset” đối với gói TCP hoặc bản tin “destination host unreachable” đối với gói UDP và ICMP. Sử dụng trong những trường hợp mình chặn họ nhưng mà mình muốn báo cho họ là t đang chặn đấy.

- LOG: Chấp nhận gói tin và có ghi lại log.

```
Tóm lại ta sẽ hiểu là: Mỗi table có nhiều chain, mỗi chain có nhiều rule, mỗi rule lại có 1 target.
Ví dụ: table filter có INPUT chain, và chain INPUT có rule là chỉ địa chỉ 1.2.3.4 mới được qua. và target là ACCEPT.
```
![](img/2021-09-05-01-38-01.png)
## 5. Các loại firewall
- IP6Tables: dùng để hạn chế các địa chỉ IPv6. ipv4 và ipv6 được lưu giữ ở các table khác nhau. do đó cần bộ rule riêng khi mà ipv6 đưọc enable. Lệnh tuơng tự như iptables
- NFTables: triển khai với các cú pháp dễ đọc hơn, hỗ trợ cả ipv4 và ipv6 cùng lúc. Mới triển khai trên các phiên bản linux mới.

## 6. Các case với iptables

**List các rule và xóa rule**
```
iptables -L -v --line-numbers
iptables -t nat -L
```
- -L list rule
- t table nào. Ví dụ: nat, mangle, raw, security. default alf filter
Xóa rule
![](img/2021-09-12-15-35-28.png)
```
iptables -n --list <name_chain> --line-numbers
iptables -D <name_chain>  1
```
- name_chain : ví dụ INPUT
- 1 : số num

**Thay đổi policy của Chain:**
```
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
```
Để chặn các packet khi không match với rule nào. -P : policy

**Thêm xóa rule: Allow packet local interface:**
```
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

iptables -D INPUT -i lo -j ACCEPT
iptables -D OUTPUT -o lo -j ACCEPT

iptables -F INPUT
iptables -F 
```
- -A append  rule vào cuối chain, -D delete rule ở đầu chain, -F: Flush  toàn bộ rule của chain.
- -i: interface. VD : eth0, lo- loopback,...
- -o: output interface
- -j: jump to target . ví dụ ACCEPT, DROP, REJECT

**Enable DNS, DHCP:**
```
iptables -A INPUT -p udp --dport 67 -j ACCEPT
iptables -A INPUT -p tcp --dport 67 -j ACCEPT
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 53 -j ACCEPT
iptables -A OUTPUT -p udp --dport 68 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 68 -j ACCEPT
```
+ Các port dịch vụ : https://cuongquach.com/download/pdf/sys/common_service_ports.pdf
+ -p: protocol giao thức cho dns, dhcp là udp, packet lớn hơn thì tcp.
+ –dport destination port, -sport source port

**Enable SSH**

*Client*:
```
iptables -A OUTPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```
- -m match : may load extension
- Tức là accept gửi đến port 22 của server. và accept nhận về các state ESTABLISHED,RELATED từ server.

*Server*
```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

**Cho phép ping đến máy khác chứ không cho bọn ló ping mình**
```
iptables -t mangle -A PREROUTING -p icmp -j DROP
iptables -A OUTPUT -p icmp -j ACCEPT
```

**Theo dõi những packet bị chặn bằng LOG**

```
iptables -N LOG_dangdh11
iptables -A LOG_dangdh11 -j LOG --log-level error --log-prefix "iptables droped: "
iptables -A LOG_dangdh11 -j DROP


iptables -A INPUT -j LOG_dangdh11
iptables -A OUTPUT -j LOG_dangdh11
```
(LOG_dangdh11: để ghi nhớ là có thể tạo tên mới tùy theo mục đích)
Thực chất là bạn có thể hiểu theo cách là : tạo hàm và gọi hàm.
- -N new chain, -X delete chain
- dòng 2-3: define ghi log và drop packet. Kiểu như là định nghĩa các chức năng trong hàm.
- dòng 4-5: add 2 chain INPUT và OUTPUT refrence đến LOG_dangdh11, ở đây sẽ define các rule để ghi lại log. Giống như việc gọi hàm

`* P/S: KHác biệt giữ DROP và REJECT: DROP thì người gửi sẽ ko biết gì, còn REJECT người gửi sẽ như là được thông báo 'connection reset by peer'.`

Các case khác:
**Limit ccu per ip**
```
iptables -A INPUT -p tcp -m connlimit --connlimit-above 100 -j REJECT --reject-with tcp-reset
```
**Limit cps per ip**
```
iptables -A INPUT -p tcp -m conntrack --ctstate NEW -m limit --limit 60/s --limit-burst 20 -j ACCEPT
iptables -A INPUT -p tcp -m conntrack --ctstate NEW -j DROP
```
**Port scanning**
```
iptables -N port_scanning
iptables -A port_scanning -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s --limit-burst 2 -j RETURN
iptables -A port_scanning -j DROP

iptable -A INPUT -j port_scanning
```
**Chặn ip theo Geolocation**
```
iptables -A droplist -i eth1 -s 1.2.3.0/24 -j LOG --log-prefix "Block IP GeoIP  "
iptables -A droplist -i eth1 -s 1.2.3.0/24 -j DROP
```
Đại khái ý tưởng là chuẩn bị những file về dải ip quốc tế (VD: db ip maxmind):
Và chuẩn bị filebash để block
```
_input=/path/to/text.txt
iptables -N block_geoip

while read -r ip
do
	iptables -A block_geoip -i eth1 -s $ip -j LOG --log-prefix "Block IP GeoIP "
	iptables -A block_geoip -i eth1 -s $ip -j DROP
done < "$_input"

iptables -I INPUT -j block_geoip
iptables -I OUTPUT -j block_geoip
iptables -I FORWARD -j block_geoip
```

## 7. Cách triển khai iptables trong thực tế
+ Viết thành script, bashshell theo mỗi trường hợp sẽ enable những chain mà mình đã chuẩn bị sẵn để xử lý tấn công, sự cố khi mà các rule L7 không chặn được.

Updating...
