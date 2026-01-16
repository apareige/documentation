# DHCP Snooping: configuration

## Kali

```bash
sudo yersinia dhcp -attack 1 -interface XX
```

```bash
sudo tcpdump -i eth0 port 67 or port 68
```

## DHCP

### Instalation de ``isc-dhcp-server``

```bash
sudo apt install isc-dhcp-server
```

### 1er fichier de configuration

```bash
sudo nano /etc/default/isc-dhcp-server
```

```bash
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#       Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#       Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="enp0s3" # ou autre interfaces du DHCP
INTERFACESv6=""
```

### Fichier de configuration pour la zone DHCP ipv4

```bash
sudo nano /etc/dhcp/dhcpd.conf 
```

```bash
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#

# option definitions common to all supported networks...
option domain-name "example.org";
option domain-name-servers #ip dns;

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
#authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;

# No service will be given on this subnet, but declaring it helps the 
# DHCP server to understand the network topology.

#subnet 10.152.187.0 netmask 255.255.255.0 {
#}

# This is a very basic subnet declaration.

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.10 192.168.1.100;
  option routers #default gateway;
}
```

## Défense contre le DHCP spoofing (sur la gig0/1)

### pasage en monde admin

```bash
en
conf t
```

### Activer le DHCP Snooping (protection contre les attaques liées au DHCP (en conf t)

```bash
ip dhcp snooping
ip dhcp snooping vlan 1
```

### Faire confiance à une interface (CELLE DU SERVEUR DHCP)

```bash
interface gigabitEthernet 0/1
ip dhcp snooping trust
```

### Protection contre le DHCP starvation (bouffe 4 adresses IP grand max)

```cisco
interface fa0/1-24 <- Configuration simultanée des 24 ports
ip dhcp snooping limit rate 4
interface gigabitEthernet 0/2
ip dhcp snooping limit rate 4
```

### Expiration de la table d'adresses MAC

```cisco
interface range [INTERFACES] aging time [MINUTES]
```

### Éteindre les interfaces lors d'une détection de violation

```cisco
switchport port-security violation shutdown
```
