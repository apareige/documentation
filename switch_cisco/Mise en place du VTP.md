# Mise en place du VTP

## Table des matières
- [Mise en place du VTP](#mise-en-place-du-vtp)
  - [Table des matières](#table-des-matières)
  - [Configuration du Switch VTP Server](#configuration-du-switch-vtp-server)
    - [Création des VLANs](#création-des-vlans)
    - [Assignation des VLANs aux interfaces](#assignation-des-vlans-aux-interfaces)
    - [Configuration du serveur VTP](#configuration-du-serveur-vtp)
  - [Configuration du Switch VTP Client](#configuration-du-switch-vtp-client)
    - [Configuration du client VTP](#configuration-du-client-vtp)

## Configuration du Switch VTP Server

### Création des VLANs

```cisco
Switch(config)# vlan X
Switch(config-vlan)# name VLAN_NAME
Switch(config-vlan)# exit
```

### Assignation des VLANs aux interfaces

Assignez les ports aux VLANs appropriés :

```cisco
Switch(config)# interface gigabitEthernet 0/X
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan X,Y,Z
Switch(config-if)# switchport nonegotiate
Switch(config-if)# switchport trunk native vlan X
Switch(config-if)# exit
```

### Configuration du serveur VTP

Configurez le switch comme serveur VTP :

```cisco
Switch(config)# vtp mode server
Switch(config)# vtp domain DOMAIN_NAME
Switch(config)# vtp password XXXXXXXXXX
Switch(config)# vtp version 2
Switch(config)# end
```

## Configuration du Switch VTP Client


### Configuration du client VTP

>[!TIPS]
> Le trunk entre le switch VTP Server et le switch VTP Client doit être configuré avant de configurer le VTP Client.



```cisco
Switch(config)# vtp mode client
Switch(config)# vtp domain XXXXXXXXXXXX
Switch(config)# vtp password XXXXXXXXXX
Switch(config)# vtp version 2
Switch(config)# end
```