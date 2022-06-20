## 初步配置
### 网关设备上关闭 firewalld 和 selinux，安装 iptables
```shell script
yum -y install iptables-services
systemctl stop firewalld 
systemctl disable firewalld 
systemctl start iptables 
systemctl enable iptables
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
### 优化内核参数
```shell script
cat >> /etc/security/limits.conf <<EOF
* - nofile 65535
* - nproc 65536
EOF

cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward = 1
net.ipv4.tcp_fin_timeout = 2
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_keepalive_time = 600
net.ipv4.ip_local_port_range = 4000 65000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.route.gc_timeout = 100
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.core.somaxconn = 16384
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_orphans = 16384
net.netfilter.nf_conntrack_max = 25000000
net.netfilter.nf_conntrack_tcp_timeout_established = 180
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
EOF

sysctl -p >/dev/null 2>&1
```

### 校准时间，设置时区
```shell script
timedatectl set-timezone Asis/Shanghai
```

## 安装 v2ray 客户端
### 使用install-release.sh脚本离线安装
```shell script
chmod + x install-release.sh
./install-release.sh -l v2ray-linux-64.zip
```
### 使用config.json将默认的配置文件替换掉
```shell script
rm -f /usr/local/etc/v2ray/config.json
cp config.json /usr/local/etc/v2ray/config.json
```

### 启动v2ray,并测试
```shell script
systemctl start v2ray
systemctl enable v2ray
systemctl status v2ray

curl -x socks5://127.0.0.1:1080 google.com
```

## 中期工作
### 重启设备并关闭 iptables
```shell script
init 6

systemctl stop iptables
```
### 在局域网内 PC 机配置 IP 地址信息，网关指向网关设备，看是否能正常上网

### 回到网关设备，开启网关设备 iptables 防火墙和配置 iptables 规则
```shell script
# 设置策略路由
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100
 
# 代理局域网设备
iptables -t mangle -N V2RAY
iptables -t mangle -A V2RAY -d 127.0.0.1/32 -j RETURN
iptables -t mangle -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY -d 66.0.19.0/24 -p tcp -j RETURN
# 直连局域网，避免 V2Ray 无法启动时无法连网关的 SSH，如果你配置的是其他网段（如 10.x.x.x 等），则修改成自己的
iptables -t mangle -A V2RAY -d 66.0.19.0/24 -p udp ! --dport 53 -j RETURN
# 直连局域网，53 端口除外（因为要使用 V2Ray 的 DNS）
iptables -t mangle -A V2RAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1
# 给 UDP 打标记 1，转发至 12345 端口
iptables -t mangle -A V2RAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
# 给 TCP 打标记 1，转发至 12345 端口
iptables -t mangle -A PREROUTING -j V2RAY
# 应用规则
 
# 代理网关本机
iptables -t mangle -N V2RAY_MASK
iptables -t mangle -A V2RAY_MASK -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_MASK -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A V2RAY_MASK -d 66.0.19.0/24 -p tcp -j RETURN
# 直连局域网，避免 V2Ray 无法启动时无法连网关的 SSH，如果你配置的是其他网段（如 10.x.x.x 等），则修改成自己的
iptables -t mangle -A V2RAY_MASK -d 66.0.19.0/24 -p udp ! --dport 53 -j RETURN
# 直连局域网，53 端口除外（因为要使用 V2Ray 的 DNS）
iptables -t mangle -A V2RAY_MASK -j RETURN -m mark --mark 0xff
# 直连 SO_MARK 为 0xff 的流量(0xff 是 16 进制数，数值上等同与上面 V2Ray 配置的 255)，此规则目的是避免代理本机(网关)流量出现回环问题
iptables -t mangle -A V2RAY_MASK -p udp -j MARK --set-mark 1
# 给 UDP 打标记 1，转发至 12345 端口
iptables -t mangle -A V2RAY_MASK -p tcp -j MARK --set-mark 1
# 给 TCP 打标记 1，转发至 12345 端口
iptables -t mangle -A OUTPUT -j V2RAY_MASK
# 应用规则
```
### 查看iptables规则命令iptables -nL -t mangle
```shell script
iptables -nL -t mangle
```
### 重启服务，回到局域网内 PC 机测试是否可以正常使用梯子
```shell script
systemctl restart v2ray && systemctl restart iptables.service
```
### 将 iptables 规则保存到 /etc/iptables/rules.v4 中
```shell script
mkdir -p /etc/iptables && iptables-save > /etc/iptables/rules.v4

```
### 在 /etc/systemd/system/ 目录下创建一个名为 tproxyrule.service 的文件，然后添加以下内容并保存
```shell script
cat > /etc/systemd/system/ <<EOF
[Unit]
Description=Tproxy rule
After=network.target
Wants=network.target

[Service]

Type=oneshot
#注意分号前后要有空格
ExecStart=/sbin/ip rule add fwmark 1 table 100 ; /sbin/ip route add local 0.0.0.0/0 dev lo table 100 ; /sbin/iptables-restore /etc/iptables/rules.v4

[Install]
WantedBy=multi-user.target
EOF
```
### 开机启动tproxyrule服务
```shell script
systemctl enable tproxyrule

```
## 优化
### 优化 v2ray 配置文件，避免出现 too many open files 问题,在 [Service] 下加入 LimitNPROC=500 和 LimitNOFILE=1000000，修改后的内容如下。
```shell script
vi /etc/systemd/system/v2ray.service
[Unit]
Description=V2Ray Service
After=network.target
Wants=network.target

[Service]
# This service runs as root. You may consider to run it as another user for security concerns.
# By uncommenting the following two lines, this service will run as user v2ray/v2ray.
# More discussion at https://github.com/v2ray/v2ray-core/issues/1011
# User=v2ray
# Group=v2ray
Type=simple
PIDFile=/run/v2ray.pid
ExecStart=/usr/bin/v2ray/v2ray -config /etc/v2ray/config.json
Restart=on-failure
# Don't restart in the case of configuration error
RestartPreventExitStatus=23
LimitNPROC=500
LimitNOFILE=1000000

[Install]
WantedBy=multi-user.target
```

### 执行 systemctl daemon-reload && systemctl restart v2ray 生效
```shell script
systemctl daemon-reload && systemctl restart v2ray
```
