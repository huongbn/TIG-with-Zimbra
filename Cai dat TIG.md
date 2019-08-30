# Hướng dẫn setup TIG Telegraf InfluxDB và Grafana monitor hệ thống mail Zimbra

# Mục lục
- [1. Cài đặt InfluxDB](#1)
- [2. Cài đặt Telegraf](#2)
- [3. Cài đặt Grafana](#3)

<a name="1"></a>

## 1. Cài đặt InfluxDB]
Thêm repositoy, thực hiện update và install 
```
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

sudo apt-get update && sudo apt-get install influxdb
sudo service influxdb start
```
- Cấu hình InfluxDB
Chỉnh sửa file config `/etc/influxdb/influxdb.conf`, bỏ comment dòng 15, 247, 256
```
# Uncomment line 15
 bind-address = "IP_zimbra:8088"
# On line 247 under http, uncomment it if you need http authentication
 [http]
 # Determines whether HTTP endpoint is enabled.
 enabled = true
# On line 256, make sure that:
 bind-address = ":8086"
```
- Enable port trên firewall
```
ufw allow 8086/tcp
ufw allow 8088/tcp
```
<a name="2"></a>

## 2. Cài đặt Telegraf]
```
apt install telegraf -y
```
- Chỉnh sửa file config `/etc/telegraf/telegraf.conf` như sau:
```
[global_tags]
# Configuration for telegraf agent
 [agent]
     interval = "10s"
     debug = false
     hostname = "server-hostname"
     round_interval = true
     flush_interval = "10s"
     flush_jitter = "0s"
     collection_jitter = "0s"
     metric_batch_size = 1000
     metric_buffer_limit = 10000
     quiet = false
     logfile = ""
     omit_hostname = false
```
- Tại sesion outputs, chỉnh sửa như sau:
```
[[outputs.influxdb]]
     urls = ["http://influxdb-ip:8086"] # Địa chỉ của InfluxDB
     database = "database-name" # Đặt tên cho DB
     timeout = "0s"
     username = "auth-username" # Xác thực username
     password = "auth-password" # Xác thực password
     retention_policy = ""
```
- Tại sesion inputs, cấu hình để thu thập các metric của hệ thống như disk, cpu, I/O, cũng như các metric ủa zimbra như mail send/recv, queue, delay...
```
[[inputs.procstat]]
   exe = "memcached"
   prefix = "memcached"
 [[inputs.procstat]]
   exe = "java"
   prefix = "java"
 [[inputs.procstat]]
   exe = "mysqld"
   prefix = "mysqld"
 [[inputs.procstat]]
   exe = "slapd"
   prefix = "slapd"
 [[inputs.procstat]]
   exe = "nginx"
   prefix = "nginx"
 [[inputs.exec]]
   commands = ["/opt/zimbra/common/bin/zimbra_pflogsumm.pl -d today /var/log/zimbra.log"]
   name_override = "zimbra_stats"
   data_format = "influx"
 [[inputs.exec]]
   commands = ["sed 's/........................//' /opt/zimbra/jetty/webapps/zimbra/downloads/.git/HEAD"]
   name_override = "zimbra_stats"
   data_format = "value"
   data_type = "string"
```
- Chạy command line sau test xem metric từ zimbra có hoạt động không
```
sudo -u telegraf /opt/zimbra/common/bin/zimbra_pflogsumm.pl /var/log/zimbra.log
```
- Restart lại telegraf
```
sudo systemctl restart telegraf
```
<a name="3"></a>

## 3. Cài đặt Grafana]
Thực hiện cài đặt Grafana
```
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
curl https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install grafana
``` 
Mở port 3000 cho grafana
```
ufw allow 3000/tcp
```
- Khởi động grafana
```
sudo service grafana-server start
sudo update-rc.d grafana-server defaults  # start with boot
```
Hoặc khởi động grafana bằng systemd
```
systemctl daemon-reload
systemctl start grafana-server
systemctl status grafana-server
sudo systemctl enable grafana-server.service
```
Giao diện dashboard Grafana như sau:



# Tham khảo
- https://wiki.zimbra.com/wiki/Monitoring_Zimbra_Collaboration_-_InfluxDB,_Telegraf_and_Grafana
- https://computingforgeeks.com/how-to-monitor-zimbra-server-with-grafana-influxdb-and-telegraf/
