# Kịch bản kiểm tra

Chạy lần lượt từng bài. Chụp màn hình kết quả để bổ sung vào README.

## 1. VLAN và trunk
Trên **SW1**:
```
show vlan brief
show interfaces trunk
```
**Mong đợi:**
- Gi0/1 nằm trong VLAN 10, Gi0/2 trong VLAN 20, Gi0/3 trong VLAN 30.
- Gi0/0 ở trạng thái **trunking**, cho phép VLAN 10, 20, 30, 99.

## 2. DHCP
Trên mỗi VPCS:
```
ip dhcp
show ip
```
**Mong đợi:** PC-SALES nhận `192.168.10.11`, gateway `192.168.10.1`, DNS `8.8.8.8`. Các PC khác tương tự với mạng của mình.

Trên **R2-HQ**:
```
show ip dhcp binding
```
**Mong đợi:** thấy danh sách IP đã cấp kèm MAC của từng PC.

## 3. Định tuyến inter-VLAN
Từ **PC-SALES**:
```
ping 192.168.20.11
```
**Mong đợi:** thành công. Gói tin đi PC-SALES → SW1 (gắn tag VLAN 10) → R2 Gi0/1.10 → R2 Gi0/1.20 → SW1 (tag VLAN 20) → PC-IT.

## 4. OSPF
Trên **R1-EDGE**:
```
show ip ospf neighbor
show ip route ospf
```
**Mong đợi:**
- 2 neighbor là `2.2.2.2` và `3.3.3.3` ở trạng thái **FULL**.
- Bảng định tuyến có các mạng `192.168.10/20/30/99/50.0/24` với ký hiệu `O`.

Trên **R2-HQ**:
```
show ip route
```
**Mong đợi:** có dòng `O*E2 0.0.0.0/0 via 10.0.12.1`. Đây là default route do R1 phát vào OSPF.

## 5. Đường dự phòng OSPF
Từ **PC-BRANCH**, ping liên tục tới PC-SALES:
```
ping 192.168.10.11 -t
```
Trên **R1-EDGE**, tắt link tới R3:
```
configure terminal
interface GigabitEthernet0/2
 shutdown
```
**Mong đợi:** mất vài gói, sau đó ping chạy lại bình thường. Lúc này R3 đi qua link dự phòng R3 → R2. Kiểm tra bằng `show ip route` trên R3. Xong thì `no shutdown` để khôi phục.

## 6. NAT ra Internet
Từ **PC-IT**:
```
ping 8.8.8.8
```
Trên **R1-EDGE**:
```
show ip nat translations
```
**Mong đợi:** ping thành công. Bảng NAT có dòng ánh xạ `192.168.20.11` thành `203.0.113.2`, kèm số port khác nhau cho từng phiên (PAT).

## 7. ACL mạng khách (GUEST-IN)
Từ **PC-GUEST**:

| Lệnh | Mong đợi | Lý do |
|---|---|---|
| `ping 8.8.8.8` | ✅ Thành công | Khách được ra Internet |
| `ping 192.168.10.11` | ❌ Thất bại | Chặn vào VLAN Sales |
| `ping 192.168.99.2` | ❌ Thất bại | Chặn vào mạng quản trị |
| `ping 192.168.50.11` | ❌ Thất bại | Chặn vào chi nhánh |
| `ping 192.168.30.1` | ❌ Thất bại | Gateway cũng nằm trong 192.168.0.0/16. Đây là hành vi chủ đích: khách không cần chạm vào router |

Trên **R2-HQ**:
```
show access-lists GUEST-IN
```
**Mong đợi:** số "matches" ở các dòng `deny` tăng lên sau mỗi lần thử.

## 8. ACL mạng quản trị (MGMT-OUT)
| Thử từ | Lệnh | Mong đợi |
|---|---|---|
| PC-IT | `ping 192.168.99.2` | ✅ Thành công |
| PC-SALES | `ping 192.168.99.2` | ❌ Thất bại |
| PC-BRANCH | `ping 192.168.99.2` | ❌ Thất bại |

## 9. Quan sát gói tin bằng Wireshark
Trong GNS3, chuột phải vào link **R2 – SW1** rồi chọn **Start capture**.
- Filter `vlan`: thấy header **802.1Q** với VLAN ID 10/20/30 trong gói ping.
- Filter `ospf`: trên link R1 – R2, thấy gói **Hello** gửi mỗi 10 giây tới `224.0.0.5`.
- Filter `bootp`: khi PC xin IP, thấy đủ 4 gói **Discover, Offer, Request, Ack**.
