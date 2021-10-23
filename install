#!/bin/bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
SYS_DT=$(date +%F-%T | tr ':' '_')

exiterr()  { echo "Error: $1" >&2; exit 1; }
exiterr2() { exiterr "'apt-get install' failed."; }
conf_bk() { /bin/cp -f "$1" "$1.old-$SYS_DT" 2>/dev/null; }
bigecho() { echo "## $1"; }


check_root() {
  if [ "$(id -u)" != 0 ]; then
    exiterr "Script must be run as root. Try 'sudo bash $0'"
  fi
}

check_vz() {
  if [ -f /proc/user_beancounters ]; then
    exiterr "OpenVZ VPS is not supported."
  fi
}

check_iptables() {
  if [ -x /sbin/iptables ] && ! iptables -nL INPUT >/dev/null 2>&1; then
    exiterr "IPTables check failed. Reboot and re-run this script."
  fi
} 

 # Update repository
  conf_bk "/etc/apt/sources.list"
cat > /etc/apt/sources.list <<EOF

deb http://archive.ubuntu.com/ubuntu focal main restricted
deb http://archive.ubuntu.com/ubuntu focal-updates main restricted
deb http://archive.ubuntu.com/ubuntu focal universe
deb http://archive.ubuntu.com/ubuntu focal-updates universe
deb http://archive.ubuntu.com/ubuntu focal multiverse
deb http://archive.ubuntu.com/ubuntu focal-updates multiverse
deb http://archive.ubuntu.com/ubuntu focal-backports main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu focal-security main restricted
deb http://archive.ubuntu.com/ubuntu focal-security universe
EOF

echo "apt-get update"
apt-get update

apt-get install -y net-tools

echo "apt install strongswan"
apt install -y strongswan


 # Create ipsec.conf
 rm /etc/ipsec.conf
  conf_bk "/etc/ipsec.conf"
cat > /etc/ipsec.conf <<EOF
config setup
conn rw-base
    fragmentation=yes
    dpdaction=clear
    dpdtimeout=90s
    dpddelay=30s

conn l2tp-vpn
    also=rw-base
    ike=aes128-sha256-modp3072
    esp=aes128-sha256-modp3072
    leftsubnet=%dynamic[/1701]
    rightsubnet=%dynamic
    mark=%unique
    leftauth=psk
    rightauth=psk
    type=transport
    auto=add
EOF

# Create ipsec.secrets
 rm /etc/ipsec.secrets
  conf_bk "/etc/ipsec.secrets"
cat > /etc/ipsec.secrets <<EOF
%any %any : PSK "1qQEuGa8N51BSNxZ"
EOF


apt install -y xl2tpd

# Create xl2tpd.conf
 rm /etc/xl2tpd/xl2tpd.conf
  conf_bk "/etc/xl2tpd/xl2tpd.conf"
cat > /etc/xl2tpd/xl2tpd.conf <<EOF
[global]                                                                ; Global parameters:
port = 1701                                                     ; * Bind to port 1701
auth file = /etc/ppp/chap-secrets
access control = no                                     ; * Refuse connections without IP match
ipsec saref = yes
force userspace = yes

[lns default]                                                   ; Our fallthrough LNS definition
exclusive = no                                          ; * Only permit one tunnel per host
ip range = 10.2.2.100-10.2.2.199        ; * Allocate from this IP range
hidden bit = no
local ip = 10.2.2.1
length bit = yes
require authentication = yes
name = l2tp-vpn
pppoptfile = /etc/ppp/options.xl2tpd
flow bit = yes
EOF

cd /etc/ppp
cp options options.xl2tpd

# Create options.xl2tpd
 rm /etc/ppp/options.xl2tpd
  conf_bk "/etc/ppp/options.xl2tpd"
cat > /etc/ppp/options.xl2tpd <<EOF
asyncmap 0

auth

crtscts

lock

hide-password

modem

mtu 1460

lcp-echo-interval 30

lcp-echo-failure 4

noipx

refuse-pap

refuse-chap

refuse-mschap

require-mschap-v2

multilink

mppe-stateful

proxyarp

ms-dns 10.2.2.1

ms-dns 8.8.8.8
EOF

systemctl restart xl2tpd

# Create chap-secrets
 rm /etc/ppp/chap-secrets
  conf_bk "/etc/ppp/chap-secrets"
cat > /etc/ppp/chap-secrets <<EOF
user    l2tp-vpn        pUmxpAw00001       *
EOF

# Create chap-secrets
 rm /etc/sysctl.d/99-sysctl.conf
  conf_bk "/etc/sysctl.d/99-sysctl.conf"
cat > /etc/sysctl.d/99-sysctl.conf<<EOF
net.ipv4.ip_forward=1
EOF

iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

sysctl -p /etc/sysctl.d/99-sysctl.conf

apt install -y iptables-persistent

systemctl restart strongswan-starter
systemctl restart xl2tpd
