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
wget --no-check-certificate https://www.privateinternetaccess.com/openvpn/openvpn-strong.zip
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
remote nl.privateinternetaccess.com 1197
resolv-retry infinite
nobind
persist-key
persist-tun
cipher aes-256-cbc
auth sha256
tls-client
remote-cert-tls server
auth-user-pass authuser
auth-nocache
comp-lzo
verb 1
reneg-sec 0
crl-verify crl.rsa.4096.pem
ca ca.rsa.4096.crt
disable-occ
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
Mon Nov 17 23:08:56 2014 OpenVPN 2.3.4 mips-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [EPOLL] [MH] [IPv6] built on Sep 20 2014
Mon Nov 17 23:08:56 2014 library versions: OpenSSL 1.0.1j 15 Oct 2014, LZO 2.08
Mon Nov 17 23:08:56 2014 UDPv4 link local: [undef]
Mon Nov 17 23:08:56 2014 UDPv4 link remote: [AF_INET]108.61.57.214:1194
Mon Nov 17 23:09:00 2014 [Private Internet Access] Peer Connection Initiated with [AF_INET]108.61.57.214:1194
Mon Nov 17 23:09:02 2014 TUN/TAP device tun0 opened
Mon Nov 17 23:09:02 2014 do_ifconfig, tt->ipv6=0, tt->did_ifconfig_ipv6_setup=0
Mon Nov 17 23:09:02 2014 /sbin/ifconfig tun0 10.198.1.10 pointopoint 10.198.1.9 mtu 1500
Mon Nov 17 23:09:02 2014 Initialization Sequence Completed
```

Check to see if tunnel interface exists (You will have to open a second SSH connection because the openvpn command above must be running):

```
ifconfig tun0
```

```
root@OpenWrt:~# ifconfig tun0
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:10.132.1.6  P-t-P:10.132.1.5  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:588 errors:0 dropped:0 overruns:0 frame:0
          TX packets:789 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100
          RX bytes:281373 (274.7 KiB)  TX bytes:159631 (155.8 KiB)
```

Close OpenVPN

```
ctrl+c
```

Force router to use privateinternetaccess.com's DNS servers:

```
uci add_list dhcp.lan.dhcp_option="6,209.222.18.222,209.222.18.218"
uci commit dhcp
```

Run VPN at startup. Go to Luci web interface, go to System -> Startup and add this before the `exit 0`:

```
openvpn --cd /etc/openvpn --config /etc/openvpn/piageneric.ovpn --remote us-east.privateinternetaccess.com 1194 &
```

Reboot for DHCP and startup changes to take effect:

```
reboot
```
