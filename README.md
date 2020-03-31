# Cài đặt và sử dụng Pfsense 

![](images/pfsense0000.png)

- VPNServer 
- Multiwan 

Mô hình cơ bản 

![](images/pfsense0001.png)

Ip Planning 

![](images/pfsense0002.png)

- Pfsense sẽ chạy trên KVM 
- Đấu nối xuống Sw-L2 chia VLAN phía dưới 
- Trên máy ảo pfsense, thực hiện đặt ip dùng là gateway cho các vlan


# Cài đặt trên KVM 

![](images/pfsense0003.gif)

Đổi IP theo IP Planning

![](images/pfsense0004.png)
![](images/pfsense0005.png)
![](images/pfsense0006.png)


# Cấu hình cơ bản 

Đăng nhập với tài khoản `admin/pfsense` và cấu hình

![](images/pfsense0007.gif)

Cấu hình `Disable checksum ofload`

![](images/pfsense0008.png)
![](images/pfsense0009.png)
![](images/pfsense0010.png)

> Mục đích là để sử dụng được với device model là “virtio”. Nếu không check vào tùy chọn này, sau này khi tạo vpn mode tap sẽ gặp lỗi máy ping được nhưng không thể ssh.

Cấu hình bổ sung `rules` cho WAN tránh bị disconnect khi thêm interface

![](images/pfsense0011.png)
![](images/pfsense0012.png)
![](images/pfsense0013.png)
![](images/pfsense0014.png)
![](images/pfsense0015.png)

Cấu hình thêm interface 
![](images/pfsense0016.gif)

Kết quả kiểm tra trên Cli
![](images/pfsense0017.png)

# Cấu hình VPN mode TAP

## Tạo Certificate

Truy cập `System/Certificate Manager/CA` -> `Add`

![](images/pfsense0018.png)
![](images/pfsense0019.png)
![](images/pfsense0020.png)

Tại tab `System/Certificate Manager/Certificate`, tạo cho server VPN

![](images/pfsense0021.png)
![](images/pfsense0022.png)
![](images/pfsense0023.png)
![](images/pfsense0024.png)

## Tạo user VPN 
Truy cập `System/UserManager`

![](images/pfsense0025.png)
![](images/pfsense0026.png)
![](images/pfsense0027.png)
![](images/pfsense0028.png)

Truy cập `System/Packet Manager` cài đặt thêm Plugin `openvpn-client-export`
![](images/pfsense0029.png)
![](images/pfsense0030.png)
![](images/pfsense0031.png)

Truy cập `VPN/OpenVPN/Server`, click `Add` để tạo VPN server
![](images/pfsense0032.png)

Khai báo các thông tin về mode kết nối:
- Server mode: Remote Access (SSL/TLS + User Auth)
- Device mode: tap
- Interface: WAN
- Local port: 1198 (tùy ý lựa chọn port)

![](images/pfsense0033.png)

Khai báo các thông tin về mã hóa
- TLS Configuration: chọn sử dụng TLS key
- Peer Certificate Authority: chọn CA cho hệ thống đã tạo trước đó (server-ca)
- Server certificate: chọn cert cho server được tạo (server-cert)
- Enable NCP: lựa chọn sử dụng mã hóa đường truyền giữa Client và Server, sử dụng các giải thuật mặc định là AES-256-GCM và  AES-128-GCM
- Auth digest algorithm: lựa chọn giải - thuật xác thực kênh truyền là SHA256


![](images/pfsense0034.png)
![](images/pfsense0035.png)

Cấu hình Tunnel như sau : 
- Bridge Interface : Chọn VLAN24, các IP của user khi VPN sẽ nhận IP dải VLAN24
- Server Bridge DHCP Start – End : Dải IP cấp cho user VPN
- Inter-client communication : Cho phép các client giao tiếp với nhau qua VPN
- Duplicate Connection : Cho phép các client cùng tên có thể kết nối VPN

![](images/pfsense0036.png)
![](images/pfsense0037.png)

Cấu hình Routing : 
- DNS Server 1 & 2 : Đặt DNS 8.8.8.8 và 8.8.4.4
- Custom option : Cho phép các dải mạng LAN được phép kết nối với nhau. 
```sh 
push "route 10.10.24.0 255.255.255.0";push "route 10.10.25.0 255.255.255.0";push "route 10.10.26.0 255.255.255.0"
```

![](images/pfsense0038.png)
![](images/pfsense0039.png)
![](images/pfsense0040.png)

Truy cập `Interfaces` cấu hình thêm Interface OpenVPN, tạo bridge mới và add 2 interface VPN và VLAN24 vào bridge

![](images/pfsense0041.gif)

Truy cập `Firewall` Bổ sung rules firewall cho VPN và lưu lại

![](images/pfsense0042.png)

Truy cập `Firewall/Rules/OPENVPN` add rule cho phép lưu lượng đi qua

![](images/pfsense0043.png)
![](images/pfsense0044.png)

Config NAT Rule : Cho phép inter VLAN, các VLAN có thể giao tiếp với nhau. 

Tại mục : `Firewall/NAT/Outbound`, chọn Add thêm NAT Rule. Chú ý chọn dạng `Hybrid`

![](images/pfsense0045.png)
![](images/pfsense0046.png)
![](images/pfsense0047.png)

Thêm Interface được NAT dải VLAN24

![](images/pfsense0048.png)

## Export OPENVPN Config và sử dụng 

Tại tab `VPN/OpenVPN/ClientExport`, khai báo các thông số:
- Remote Access Server: lựa chọn OpenVPN server và port 1198
- Hostname Resolution: lựa chọn khai báo Interface IP của WAN 

![](images/pfsense0049.png)

Lựa chọn và tải config về máy

![](images/pfsense0050.png)

Tiến hành vpn và xem ip được nhận, ping thử các host cùng dải.
