# Switch Snooping: configuration
- [Switch Snooping: configuration](#switch-snooping-configuration)
- [Défense contre le DHCP spoofing (sur la gig0/1)](#défense-contre-le-dhcp-spoofing-sur-la-gig01)
  - [pasage en monde admin](#pasage-en-monde-admin)
  - [Activer le DHCP Snooping (protection contre les attaques liées au DHCP (en conf t)](#activer-le-dhcp-snooping-protection-contre-les-attaques-liées-au-dhcp-en-conf-t)
  - [Faire confiance à une interface (CELLE DU SERVEUR DHCP)](#faire-confiance-à-une-interface-celle-du-serveur-dhcp)
  - [Protection contre le DHCP starvation (bouffe 4 adresses IP grand max)](#protection-contre-le-dhcp-starvation-bouffe-4-adresses-ip-grand-max)
  - [Expiration de la table d'adresses MAC](#expiration-de-la-table-dadresses-mac)
  - [Éteindre les interfaces lors d'une détection de violation](#éteindre-les-interfaces-lors-dune-détection-de-violation)


---

# Défense contre le DHCP spoofing (sur la gig0/1)

## pasage en monde admin

```bash
en
conf t
```

## Activer le DHCP Snooping (protection contre les attaques liées au DHCP (en conf t)

```bash
ip dhcp snooping
ip dhcp snooping vlan 1
```

## Faire confiance à une interface (CELLE DU SERVEUR DHCP)

```bash
interface gigabitEthernet 0/1
ip dhcp snooping trust
```

## Protection contre le DHCP starvation (bouffe 4 adresses IP grand max)

```cisco
interface fa0/1-24 <- Configuration simultanée des 24 ports
ip dhcp snooping limit rate 4
interface gigabitEthernet 0/2
ip dhcp snooping limit rate 4
```

## Expiration de la table d'adresses MAC

```cisco
interface range [INTERFACES] aging time [MINUTES]
```

## Éteindre les interfaces lors d'une détection de violation

```cisco
switchport port-security violation shutdown
```
