# Khắc phục lỗi UFW không hoạt động khi sử dụng Docker
***Khóa truy cập thông qua địa chỉ ip:port cụ thể, hiện tại dùng ufw để block port 19999 nhưng khi truy cập qua ip của vps dưới dạng x.x.x.x:19999 thì vẫn vào được trang.
UFW không chặn được truy cập các port được khai báo bởi Docker.***
## Nguyên nhân gây lỗi
- Nguyên nhân khiến UFW không còn hoạt động là do Docker có quyền chỉnh sửa trực tiếp vào iptables của hệ thống.  Khi kích hoạt container service, Docker tự động thêm Rule DNAT vào iptables, giúp truy cập được cổng của container từ bên ngoài.

	- Kiểm tra iptables bằng lệnh sau:
	``` 
	iptables -t nat --list | grep _port
	```


## Cách khắc phục
### 1. Ngăn Docker chỉnh sửa iptables
- Chỉnh sửa file `/etc/docker/deamon.json`
	```
	sudo nano /etc/docker/daemon.json
	```
- Thêm vào dòng `{ "iptables" : false }` và lưu lại.
- Khởi động lại Docker bằng lệnh
	``` 
	sudo service docker restart 
	```

- Kiểm tra lại kết nối từ bên ngoài, giờ đã yên tâm _port đã bị khóa.
	```
	Starting Nmap 7.93 ( https://nmap.org ) at 2022-11-27 16:27 +07
	Nmap scan report for static.xxx.xxx.xxxclients.your-server.de (xxx.xxx.xxx.xxx)
	Host is up.

	PORT     STATE    SERVICE
	8888/tcp filtered http-proxy

	Nmap done: 1 IP address (1 host up) scanned in 2.04 seconds
	```

	#### Ưu và nhược điểm của cách này như sau:

	- Ưu điểm: chỉnh sửa nhanh, gọn, lẹ, phù hợp với các máy chủ chỉ chạy 1-2 dịch vụ Docker.
	- Nhược điểm: tính năng quản lý mạng của Docker bị vô hiệu hóa, khiến cho các dịch vụ mạng khởi tạo sau đó sẽ không thể truy cập được mạng Internet.


		Nếu muốn dịch vụ Docker truy cập được mạng Internet, phải chỉnh sửa file cấu hình của UFW:

		```
		 $ nano /etc/default/ufw 
		```

		Thêm vào dòng sau để mở kết nối mạng cho Docker network:
		```
		$ -A POSTROUTING ! -o docker0 -s 172.17.0.0/16 -j MASQUERADE
		```
		*Trong đó 172.17.0.0/16 là Subnet của Docker network. Nếu có nhiều Docker network, phải thêm nhiều dòng tương ứng.*

### 2. Sử dụng ufw-docker

- Để vừa bảo đảm các rule tường lửa của UFW hoạt động trơn tru, vừa giúp Docker duy trì tính năng quản lý mạng, chúng ta sẽ nhờ đến sự trợ giúp của công cụ `ufw-docker` [link](https://github.com/chaifeng/ufw-docker).

	#### Install on ubuntu
	```
	$ sudo wget -O /usr/local/bin/ufw-docker \
	  https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker
	```
	```
	$ sudo chmod +x /usr/local/bin/ufw-docker
	```

	Cài đặt `ufw-docker` bằng lệnh sau, nó sẽ thêm một số rule vào file `after.rules` của `ufw`
	```
	 $ ufw-docker install 
	```

	Để mở truy cập vào tất cả các cổng đang mở của service, mình dùng lệnh sau

	```
	ufw-docker allow [port || container name]
	```

	Kiểm tra status

	` ufw-docker status `


*Vậy là xong. UFW và Docker giờ đã hoạt động êm ái cùng nhau.*




