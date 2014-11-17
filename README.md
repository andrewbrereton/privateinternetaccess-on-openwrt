# OpenVPN on OpenWrt Barrier Breaker

Steps stolen from [Logan Marchione's blog post](http://www.loganmarchione.com/2014/10/openwrt-with-openvpn-client-on-tp-link-tl-mr3020/).

```
ssh root@192.168.1.1
```

Install packages:

```
opkg update
opkg install openvpn-openssl wget unzip
```

Create a new interface for the VPN:

```
cat >> /etc/config/network << EOF
config interface 'PIA_VPN'
    option proto 'none'
    option ifname 'tun0'
EOF
```

Download OpenVPN config from privateinternetaccess.com:

```
cd /etc/openvpn
wget --no-check-certificate https://www.privateinternetaccess.com/openvpn/openvpn.zip
unzip openvpn.zip
rm openvpn.zip
```

Create file with your privateinternetaccess.com credentials:

```
cat >> /etc/openvpn/authuser << EOF
$username
$password
EOF
```

Set permissions on authuser file:

```
chmod 400 /etc/openvpn/authuser
```

Create a generic OpenVPN config:

```
cat >> /etc/openvpn/piageneric.ovpn << EOF
client
dev tun
proto udp
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
tls-client
remote-cert-tls server
auth-user-pass authuser
auth-nocache
comp-lzo
verb 1
reneg-sec 0
crl-verify crl.pem
keepalive 10 120
EOF
```

Create firewall zone for new VPN connection:

```
cat >> /etc/config/firewall << EOF
config zone
    option name 'VPN_FW'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'
    option masq '1'
    option mtu_fix '1'
    option network 'PIA_VPN'
 
config forwarding                               
        option dest 'VPN_FW'                    
        option src 'lan' 
EOF
```

Reboot:

```
reboot
```

Log back in:

```
ssh root@192.168.1.1
```

Start the VPN:

```
openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-east.privateinternetaccess.com 1194
```

Confirm that output looks something like this:

```
root@OpenWrt:~# openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-east.privateinternetaccess.com 1194
Thu Nov 13 21:21:00 2014 OpenVPN 2.3.4 mips-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [MH] [IPv6] built on Sep 20 2014
Thu Nov 13 21:21:00 2014 library versions: OpenSSL 1.0.1j 15 Oct 2014, LZO 2.08
Thu Nov 13 21:21:00 2014 WARNING: file 'authuser' is group or others accessible
Thu Nov 13 21:21:00 2014 UDPv4 link local: [undef]
Thu Nov 13 21:21:00 2014 UDPv4 link remote: [AF_INET]216.155.129.59:1194
Thu Nov 13 21:21:06 2014 [Private Internet Access] Peer Connection Initiated with [AF_INET]216.155.129.59:1194
Thu Nov 13 21:21:09 2014 TUN/TAP device tun0 opened
Thu Nov 13 21:21:09 2014 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Thu Nov 13 21:21:09 2014 /sbin/ifconfig tun0 10.137.1.6 pointopoint 10.137.1.5 mtu 1500
Thu Nov 13 21:21:09 2014 Initialization Sequence Completed
```

Check to see if tunnel interface exists:

```
ifconfig tun0
```

Force router to use privteinternetaccess.com's DNS servers:

```
uci add_list dhcp.lan.dhcp_option="6,209.222.18.222,209.222.18.218"
uci commit dhcp
reboot
```

Run VPN at startup. Go to Luci web interface, go to System -> Startup and add this before the `exit 0`:

```
openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-east.privateinternetaccess.com 1194 &
```

