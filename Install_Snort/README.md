# Mô hình


<p align="center">
  <img src="../images/topo.png">
</p>


# Cài đặt
Cài đặt và cấu hình Snort phiên bản 2.9.x trên CentOS 8.
Cấu hình chế độ Inline cho Snort để chạy Snort như một NIPS.
Các package được cài đặt kèm theo:
- <b><u>Snort</b></u>: NIPS.
- <b><u>Barnyard2</b></u>: Phần mềm lấy output của Snort và ghi vào CSLD. Ở đây mình sẽ lưu vào hệ quản trị csdl MySql.
- <b><u>PulledPork</b></u>: tự động tải các Snort rule miễn phí mới nhất.
- <b><u>BASE</b></u>: giao diện đồ họa nền web viết bằng php, dùng để xem các Snort event.

## Kịch bản
Cấu hình NIPS sử dụng Snort để bảo vệ vùng DMZ của doanh nghiệp khỏi các tác nhân gây hại đi vào vùng này. Cấu hình Snort bảo vệ server trước cuộc tấn công "Ping of death". Cài đặt và thử nghiệm một số package của Snort. 

## <div id="install_snort">Cài đặt Snort</div>
*Lưu ý*: đảm bảo IPS phải kết nối được Internet
### <u>Bước 1: Cập nhập hệ thống.</u>
Cập nhập lại máy CentOS8 để đảm có thể chạy được các module mới nhất.
```sh
dnf update -y
```
Cài đặt thêm "epel-release" và "ethtool"
```sh
dnf install epel-release -y
dnf install ethtool -y
```

**NOTE**: Một số network card có tính năng gọi là "Large Receive Offload" (lro) và "Generic Receive Offload" (gro). khi kích hoạt tính năng này, network card thực hiện lắp ráp lại packet trước khi chúng được xử lý bởi kernel. mặc định, Snort sẽ xóa các packet lớn hơn default snaplen là 1518 bytes. Thêm vào đó, LRO, GRO có thể là nguyên nhân của vấn đề Stream5 target-based reassembly. Vì thế, nên tắt LRO và GRO.

Để tắt LRO và GRO ta sử dụng lệnh ethtool. Thêm 02 dòng sau vào mỗi interface.

```sh
ethtool -K $INTERFACE gro off
ethtool -K $INTERFACE lro off
```

Reset lại interface.
```sh
nmcli connection down $INTERFACE && nmcli connection up $INTERFACE
```


### <u>Bước 2: Cài đặt các thư viện cần thiết.</u>
Cài đặt các thư viên cần thiết để Snort hoạt động:
- pcap
- PCRE
- Libdnet
- DAQ

Cài đặt thêm một số công cụ cần thiết.
```sh
dnf install  gcc flex bison zlib* libxml2 libpcap* pcre* tcpdump git libtool curl libdnet -y
dnf --enablerepo=powertools install libdnet-devel -y
dnf --enablerepo=powertools install libpcap-devel -y

# install daq
wget https://www.snort.org/downloads/snort/daq-2.0.7.tar.gz
tar -xvzf daq-2.0.7.tar.gz
cd daq-2.0.7
./configure
sudo make & make install
```
<!-- dnf groupinstall "Development Tools" -y # Không cần thiết -->
Cập nhật các đường dẫn thư viện:
```sh
echo /usr/lib >> /etc/ld.so.conf 
echo /usr/local/lib >> /etc/ld.so.conf
```

Tạo sysmbol link cho libdnet
```sh
ln -s /usr/lib64/libdnet.so.1.0.1 /lib64/libdnet.1
```



### <u>Bước 3: Cài đặt Snort.</u>
- Cách 1:
```sh
dnf install https://www.snort.org/downloads/snort/snort-2.9.18-1.centos8.x86_64.rpm -y
```

- Cách 2:
```sh
wget https://www.snort.org/downloads/snort/snort-2.9.18-1.centos8.x86_64.rpm
dnf install snort-2.9.18-1.centos8.x86_64.rpm -y
```


- Cách 3:
```sh
cd ~/snort_src
wget https://www.snort.org/downloads/snort/snort-2.9.18.tar.gz
tar -xzvf snort-2.9.18.tar.gz
cd snort-2.9.18.tar.gz
./configure --enable-sourcefire
make
make install
```

Kiểm tra lại version của snort.

```sh
snort -V
```
**Lưu ý**: Link tải trên có thể bị thay đổi trong tương lai, kiểm tra link tải còn hoạt động không trước khi chạy các lệnh trên. Nếu không hợp lệ nữa thì thay thế bằng link hợp lệ.

### <u>Bước 4: Tạo các cấu trúc lưu trữ cho Snort.</u>
Tạo các thư mục cấu trúc cho việc lưu cấu hình Snort.
```sh
mkdir -p /etc/snort/rules
mkdir -p /var/log/snort
mkdir -p /usr/local/lib/snort_dynamicrules
```

Tạo các file cần thiết.
```sh
touch /etc/snort/rules/white_list.rules
touch /etc/snort/rules/black_list.rules
touch /etc/snort/rules/local.rules
touch /etc/snort/rules/snort.rules
touch /var/log/snort/snort.log
```

Phân quyền cho các thư mục.
```sh
chmod -R 5775 /etc/snort
chmod -R 5775 /var/log/snort
chmod -R 5775 /usr/local/lib/snort_dynamicrules

chown -R snort:snort /etc/snort
chown -R snort:snort /var/log/snort
chown -R snort:snort /usr/local/lib/snort_dynamicrules
```
### <u>Bước 5: Cấu hình Snort chạy ở chế độ NIDS.</u>

Chỉnh sửa lại file cấu hình Snort (/etc/snort/snort.conf).
```sh
sed -i 's/include \$RULE\_PATH/#include \$RULE\_PATH/' /etc/snort/snort.conf
vi /etc/snort/snort.conf
```
 - Chỉnh sửa lại dòng 45. "any" thành địa chỉ vùng mạng muốn bảo vệ.
```sh
ipvar HOME_NET 192.168.1.0/24
```

- Chỉnh sửa lại dòng 48.
```sh
 ipvar EXTERNAL_NET !$HOME_NET
```
- Chỉnh sửa lại dòng 105 - 106 và 113 - 114.
```sh
var SO_RULE_PATH /etc/snort/so_rules
var PREPROC_RULE_PATH /etc/snort/preproc_rules
...
var WHITE_LIST_PATH /etc/snort/rules
var BLACK_LIST_PATH /etc/snort/rules
```
- Chỉnh sửa dòng 521, 546, 547.
```sh
 output unified2: filename snort.log, limit 128
 ...
 include $RULE_PATH/local.rules
 include $RULE_PATH/snort.rules
```
Kiểm tra tập tin cấu hình snort. Nếu ra "<u>Snort successfully validated the configuration!</u>" là đã thành công.
```sh
snort -T -c /etc/snort/snort.conf
```
### <u>Bước 6: Cấu hình chế độ Inline và NIPS cho Snort.</u>

Chỉnh sửa file cấu hình
```sh
vi /etc/snort/snort.conf
```

- Thêm vào dòng 188.
```sh
config policy_mode:inline
```

- Chỉnh sửa từ dòng 159 đến dòng 162

```sh
config daq: afpacket
onfig daq_dir: /usr/local/lib/daq
config daq_mode: inline
config daq_var: buffer_size_mb=1024
```
Kiểm tra tập tin cấu hình snort. Nếu ra "<u>Snort successfully validated the configuration!</u>" là đã thành công.
```sh
snort -c /etc/snort/snort.conf -i ens33:ens34 -T -Q
```
**Lưu ý**: option -i các bạn thay thế 2 interface thích hợp của máy vào.

## Thực nghiệm, kiểm tra hoạt động của Snort.
Chúng ta tiến hành ping liên tục với khối dữ liệu lớn vào server. tháy thế $IP_SERVER bằng địa chỉ IPv4 của server.
```sh
ping $IP_SERVER -l 20000
```

Truy cập tập tin local.rules.
```sh
vi /etc/snort/rules/local.rules
```
Thêm nội dung rule vào tập tin
```sh
drop icmp any any -> $HOME_NET any (msg:"--> Ping of death attack!"; dsize:>10000; gid:1000001; sid:1000001;rev:1;)
```
Chạy lệnh giam sát snort trên console và theo dõi
```sh
snort -c /etc/snort/snort.conf -i ens33:ens34 -A console -Q -u snort -g snort -q
```
<p align="center">
  <img height="400" src="../images/drop_pingofdeath.png">
</p>

Ngăn chặn thành công cuộc tấn công "Ping of death".

## Cài đặt "PulledPork" cho Snort.
## Cài đặt "Barnyard2" cho Snort.
## Cài đặt "BASE" cho Snort.



Thế là đã ngăn chặn được cuộc tấn cống "Ping of death" tới server
# Tham khảo
- <a>https://programmersought.com/article/61256098977/</a>

