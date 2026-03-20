# NAT et PAT — Commandes Cisco

## Table des matières
1. [Définir les interfaces NAT](#1-définir-les-interfaces-nat)
2. [NAT statique](#2-nat-statique)
3. [NAT dynamique](#3-nat-dynamique)
4. [PAT (NAT dynamique avec surcharge)](#4-pat-nat-dynamique-avec-surcharge)

---

## 1. Définir les interfaces NAT

Indiquer au routeur quelles interfaces sont côté réseau **privé** (`inside`) et côté réseau **public** (`outside`).

    R1(config)# int fa0/0
    R1(config-if)# ip nat inside
    R1(config-if)# exit
    R1(config)# int fa0/1
    R1(config-if)# ip nat inside
    R1(config-if)# exit
    R1(config)# int s0/0
    R1(config-if)# ip nat outside
    R1(config-if)# exit

---

## 2. NAT statique

Exposer une machine interne sur une adresse publique fixe (« ouvrir un port »). Tout ce qui arrive sur `s0/0` à destination de `<ip-publique>` est redirigé vers `<ip-privée>`.

    R1(config)# ip nat inside source static <ip-privée> <ip-publique>

**Exemple :** `ip nat inside source static 192.168.1.100 201.49.10.30`

**Vérification :**

    R1# show ip nat translations

| Pro | Inside global | Inside local  | Outside local | Outside global |
|-----|---------------|---------------|---------------|----------------|
| --- | 201.49.10.30  | 192.168.1.100 | ---           | ---            |

---

## 3. NAT dynamique

Plusieurs machines privées partagent un **pool** d'adresses publiques (une adresse publique par machine, sans surcharge).

**1. Créer le pool d'adresses publiques**

    R1(config)# ip nat pool <nom-pool> <ip-début> <ip-fin> netmask <masque>

Exemple : `ip nat pool POOL-NAT-LAN2 201.49.10.17 201.49.10.30 netmask 255.255.255.240`

**2. Créer une ACL pour définir les sources à translater**

    R1(config)# access-list <numéro> deny <ip-exclue>
    R1(config)# access-list <numéro> permit <réseau> <wildcard>

Exemple : `access-list 1 deny 192.168.1.100` puis `access-list 1 permit 192.168.1.0 0.0.0.255`

**3. Appliquer le NAT dynamique**

    R1(config)# ip nat inside source list <numéro-acl> pool <nom-pool>

> Si le nombre de machines dépasse le nombre d'adresses publiques, ajouter `overload` pour basculer en PAT automatiquement.

---

## 4. PAT (NAT dynamique avec surcharge)

Configuration la plus courante (réseau domestique). Toutes les machines partagent **une seule adresse publique** — celle de l'interface `serial 0/0` — en multiplexant les ports.

**1. Créer une ACL**

    R1(config)# access-list <numéro> permit <réseau> <wildcard>

Exemple : `access-list 2 permit 192.168.0.0 0.0.0.255`

**2. Appliquer le PAT sur l'interface publique**

    R1(config)# ip nat inside source list <numéro-acl> interface serial 0/0 overload

Exemple : `ip nat inside source list 2 interface serial 0/0 overload`

> Le routeur remplace l'IP source par celle de `serial 0/0` et utilise les numéros de ports pour distinguer les sessions (`overload` = PAT).