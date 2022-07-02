# 全球加速客户端VPN搭建
## 基于Openswan on Amazon Linux 2，AWS Global Accelerator

### 安装epel源
为什么要安装epel源呢？是因为必要组件xl2tpd在基础的yum源里面是没有的。
```
sudo su
amazon-linux-extras install epel
```

### 安装依赖组件
安装完epel源以后就可以直接安装依赖组件了。

```
yum install -y openswan ppp pptpd xl2tpd wget
```

### 修改配置文件
需要等待所有依赖组件安装完成才能执行以下步骤（小标题括号内是文件路径）。

### ipsec.conf配置文件
```
# /etc/ipsec.conf - Libreswan IPsec configuration file
# This file:  /etc/ipsec.conf
#
# Enable when using this configuration file with openswan instead of libreswan
#version 2
#
# Manual:     ipsec.conf.5
# basic configuration
config setup
    # NAT-TRAVERSAL support, see README.NAT-Traversal
    nat_traversal=yes
    # exclude networks used on server side by adding %v4:!a.b.c.0/24
    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12
    # OE is now off by default. Uncomment and change to on, to enable.
    oe=off
    # which IPsec stack to use. auto will try netkey, then klips then mast
    protostack=netkey
    force_keepalive=yes
    keep_alive=1800
conn L2TP-PSK-NAT
    rightsubnet=vhost:%priv
    also=L2TP-PSK-noNAT
conn L2TP-PSK-noNAT
    authby=secret
    pfs=no
    auto=add
    keyingtries=3
    rekey=no
    ikelifetime=8h
    keylife=1h
    type=transport
    left=$serverip
    leftid=$serverip
    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any
    dpddelay=40
    dpdtimeout=130
    dpdaction=clear
# For example connections, see your distribution's documentation directory,
# or the documentation which could be located at
#  /usr/share/docs/libreswan-3.*/ or look at https://www.libreswan.org/
#
# There is also a lot of information in the manual page, "man ipsec.conf"
# You may put your configuration (.conf) file in the "/etc/ipsec.d/" directory
# by uncommenting this line
#include /etc/ipsec.d/*.conf
```

### 设置预共享密钥配置文件
```
#include /etc/ipsec.d/*.secrets
$serverip username PSK password
```
注解：第二行中username为登录名，password为登录密码

### pptpd.conf配置文件
```
#ppp /usr/sbin/pppd
option /etc/ppp/options.pptpd
#debug
# stimeout 10
#noipparam
logwtmp
#vrf test
#bcrelay eth1
#delegate
#connections 100
localip 10.0.1.2
remoteip 10.0.1.200-254
```

### xl2tpd.conf配置文件
```
;
; This is a minimal sample xl2tpd configuration file for use
; with L2TP over IPsec.
;
; The idea is to provide an L2TP daemon to which remote Windows L2TP/IPsec
; clients connect. In this example, the internal (protected) network
; is 192.168.1.0/24.  A special IP range within this network is reserved
; for the remote clients: 192.168.1.128/25
; (i.e. 192.168.1.128 ... 192.168.1.254)
;
; The listen-addr parameter can be used if you want to bind the L2TP daemon
; to a specific IP address instead of to all interfaces. For instance,
; you could bind it to the interface of the internal LAN (e.g. 192.168.1.98
; in the example below). Yet another IP address (local ip, e.g. 192.168.1.99)
; will be used by xl2tpd as its address on pppX interfaces.
[global]
; ipsec saref = yes
listen-addr = 104.171.165.91
auth file = /etc/ppp/chap-secrets
port = 1701
[lns default]
ip range = 10.0.1.100-10.0.1.254
local ip = 10.0.1.1
refuse chap = yes
refuse pap = yes
require authentication = yes
name = L2TPVPN
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```

### options.pptpd配置文件
```
# Authentication
name pptpd
#chapms-strip-domain
# Encryption
# BSD licensed ppp-2.4.2 upstream with MPPE only, kernel module ppp_mppe.o
# {{{
refuse-pap
refuse-chap
refuse-mschap
# Require the peer to authenticate itself using MS-CHAPv2 [Microsoft
# Challenge Handshake Authentication Protocol, Version 2] authentication.
require-mschap-v2
# Require MPPE 128-bit encryption
# (note that MPPE requires the use of MSCHAP-V2 during authentication)
require-mppe-128
# }}}
# OpenSSL licensed ppp-2.4.1 fork with MPPE only, kernel module mppe.o
# {{{
#-chap
#-chapms
# Require the peer to authenticate itself using MS-CHAPv2 [Microsoft
# Challenge Handshake Authentication Protocol, Version 2] authentication.
#+chapms-v2
# Require MPPE encryption
# (note that MPPE requires the use of MSCHAP-V2 during authentication)
#mppe-40    # enable either 40-bit or 128-bit, not both
#mppe-128
#mppe-stateless
# }}}
ms-dns 8.8.4.4
ms-dns 8.8.8.8
#ms-wins 10.0.0.3
#ms-wins 10.0.0.4
proxyarp
#10.8.0.100
# Logging
#debug
#dump
lock
nobsdcomp 
novj
novjccomp
nologfd
```

### options.xl2tpd配置文件
```
rm -f /etc/ppp/options.xl2tpd
cat >>/etc/ppp/options.xl2tpd<<EOF
#require-pap
#require-chap
#require-mschap
ipcp-accept-local
ipcp-accept-remote
require-mschap-v2
ms-dns 8.8.8.8
ms-dns 8.8.4.4
asyncmap 0
auth
crtscts
lock
hide-password
modem
debug
name l2tpd
proxyarp
lcp-echo-interval 30
lcp-echo-failure 4
mtu 1400
noccp
connect-delay 5000
# To allow authentication against a Windows domain EXAMPLE, and require the
# user to be in a group "VPN Users". Requires the samba-winbind package
# require-mschap-v2
# plugin winbind.so
# ntlm_auth-helper '/usr/bin/ntlm_auth --helper-protocol=ntlm-server-1 --require-membership-of="EXAMPLE\VPN Users"'
# You need to join the domain on the server, for example using samba:
# http://rootmanager.com/ubuntu-ipsec-l2tp-windows-domain-auth/setting-up-openswan-xl2tpd-with-native-windows-clients-lucid.html
```

### 创建chap-secrets配置文件，即用户列表及密码
```
# Secrets for authentication using CHAP
# client     server     secret               IP addresses
username          pptpd     password               *
username          l2tpd     password               *
```

注解：第三第四行中username为登录名，password为登录密码

### 系统配置
### 允许IP转发
```
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.default.rp_filter=0
sysctl -w net.ipv4.conf.$eth.rp_filter=0
sysctl -w net.ipv4.conf.all.send_redirects=0
sysctl -w net.ipv4.conf.default.send_redirects=0
sysctl -w net.ipv4.conf.all.accept_redirects=0
sysctl -w net.ipv4.conf.default.accept_redirects=0
```

注解：以上均是命令，复制上去运行即可
也可以修改配置文件( /etc/sysctl.conf)：

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.$eth.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
```

启动并设置开机自启动服务
```
systemctl enable pptpd ipsec xl2tpd
systemctl restart pptpd ipsec xl2tpd
```

