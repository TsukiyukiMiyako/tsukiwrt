# rax3000m nand版本openwrt校园网防检测固件
release下载固件刷入

默认ip：203.0.113.1

账号：root

密码：无

核心插件：rkp-ipid、ua2f、ipopt

## ua2f设置

```
# 启用 UA2F
uci set ua2f.enabled.enabled=1

# 可选的防火墙配置选项
# 自动添加防火墙规则
uci set ua2f.firewall.handle_fw=1

# 是否尝试处理 443 端口的流量， 通常来说，流经 443 端口的流量是加密的，因此无需处理
# uci set ua2f.firewall.handle_tls=1

# 是否处理微信的流量，微信的流量通常是加密的，因此无需处理。这一规则在启用 nftables 时无效
# uci set ua2f.firewall.handle_mmtls=1

# 是否处理内网流量，如果你的路由器是在内网中，且你想要处理内网中的流量，那么请启用这一选项
uci set ua2f.firewall.handle_intranet=1

# 使用自定义 User-Agent
uci set ua2f.main.custom_ua="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36 Edg/122.0.0.0"

# 禁用 Conntrack 标记，这会降低性能，但是有助于和其他修改 Connmark 的软件共存
uci set ua2f.main.disable_connmark=1

# 应用配置
uci commit ua2f

# 开机自启
service ua2f enable

# 启动 UA2F
service ua2f start

# 读取日志
logread | grep UA2F
```

## iptables设置
```
#修改ttl
iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64

# 防时钟偏移检测
iptables -t nat -N ntp_force_local
iptables -t nat -I PREROUTING -p udp --dport 123 -j ntp_force_local
iptables -t nat -A ntp_force_local -d 0.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 127.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 203.0.113.0/24 -j RETURN
iptables -t nat -A ntp_force_local -s 203.0.113.0/24 -j DNAT --to-destination 203.0.113.1

# 通过 rkp-ipid 设置 IPID
iptables -t mangle -N IPID_MOD
iptables -t mangle -A FORWARD -j IPID_MOD
iptables -t mangle -A OUTPUT -j IPID_MOD
iptables -t mangle -A IPID_MOD -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 203.0.113.0/24 -j RETURN
iptables -t mangle -A IPID_MOD -d 255.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -j MARK --set-xmark 0x10/0x10

# iptables 拒绝 AC 进行 Flash 检测
iptables -I FORWARD -p tcp --sport 80 --tcp-flags ACK ACK -m string --algo bm --string " src=\"http://1.1.1." -j DROP

#劫持dns
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53

#ua2f
iptables -t mangle -A ua2f -d 203.0.113.0/24 -j RETURN


#以下自动处理了，不加入
# iptables -t mangle -N ua2f
# iptables -t mangle -A ua2f -d 127.0.0.0/8 -j RETURN
#iptables -t mangle -A ua2f -p tcp --dport 443 -j RETURN
#iptables -t mangle -A ua2f -p tcp --dport 22 -j RETURN # 不处理 SSH 和 https
#iptables -t mangle -A ua2f -p tcp --dport 80 -j CONNMARK --set-mark 44
#iptables -t mangle -A ua2f -m connmark --mark 43 -j RETURN # 不处理标记为非 http 的流 (实验性)
#iptables -t mangle -A ua2f -m set --set nohttp dst,dst -j RETURN
#iptables -t mangle -A ua2f -p tcp --dport 80 -m string --string "/mmtls/" --algo bm -j RETURN # 不处理微信的 mmtls
#iptables -t mangle -A ua2f -j NFQUEUE --queue-num 10010
#iptables -t mangle -A FORWARD -p tcp -m conntrack --ctdir ORIGINAL -j ua2f

```
