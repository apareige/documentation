# ACL – Access Control List (Cisco IOS)

## Table des matières
1. [Généralités](#généralités)
2. [Types d'ACL](#types-dacl)
3. [ACL Standard](#acl-standard)
4. [ACL Étendue](#acl-étendue)
5. [Modifier une ACL](#modifier-une-acl)
6. [Supprimer une ACL](#supprimer-une-acl)
7. [Appliquer une ACL](#appliquer-une-acl)
8. [Vérification](#vérification)

---

## Généralités

- Filtre le trafic réseau selon des critères : IP source/destination, port, protocole.
- Actions possibles : `permit` (autoriser) ou `deny` (bloquer).
- Maximum **1 ACL par interface et par sens** (in / out).
- Analyse **séquentielle** : dès qu'une règle correspond, elle s'applique — le reste est ignoré.
- **Implicit deny** : tout trafic ne correspondant à aucune règle est rejeté par défaut.

> **Règles de placement :**
> - ACL Standard → au plus proche de la **destination**
> - ACL Étendue → au plus proche de la **source**
> - Règles les plus précises **en tête** de liste

---

## Types d'ACL

| Type | Plage numérique | Critères d'analyse |
|---|---|---|
| Standard | 1–99 / 1300–1999 | IP source uniquement |
| Étendue | 100–199 / 2000–2699 | IP src/dst, protocole, ports |
| Nommée | — (chaîne alphanum.) | Identique selon le type choisi |

---

## ACL Standard

### Numérique

    access-list <numéro_1-99> permit|deny <ip_source> <wildcard>
    access-list <numéro_1-99> permit any

### Nommée

    ip access-list standard <nom_acl>
     permit <réseau> <wildcard>
     deny   <réseau> <wildcard>
     permit any
     exit

---

## ACL Étendue

### Format général d'une règle

    access-list <numéro_100-199> permit|deny <protocole> <ip_src> <wildcard_src> <ip_dst> <wildcard_dst> [eq <port>]

| Champ | Valeurs possibles |
|---|---|
| `<protocole>` | `tcp`, `udp`, `icmp`, `ip` (tous) |
| `<ip_src/dst>` | `<réseau> <wildcard>` / `host <ip>` / `any` |
| `[eq <port>]` | `80`, `443`, `www`, `22`, etc. |

### Numérique

    access-list <numéro> permit tcp any host <ip_destination> eq <port>
    access-list <numéro> permit icmp <réseau_src> <wildcard> host <ip_destination>

### Nommée

    ip access-list extended <nom_acl>
     permit tcp any host <ip_destination> eq <port>
     permit icmp <réseau_src> <wildcard> host <ip_destination>
     exit

---

## Modifier une ACL

    ip access-list standard <numéro_ou_nom>
     no <numéro_séquence>
     <nouveau_numéro_séquence> permit|deny <réseau> <wildcard>

- `no <numéro_séquence>` : supprime la règle portant ce numéro de séquence.
- Insérer une règle entre deux existantes en choisissant un numéro intermédiaire.

> **Conseil :** désactiver l'ACL sur l'interface avant modification, puis la réappliquer.

---

## Supprimer une ACL

    no access-list <numéro>

Supprime une ACL numérotée (toutes ses règles).

    no ip access-list standard|extended <nom_ou_numéro>

Supprime une ACL nommée ou numérotée via le mode nommé.

---

## Appliquer une ACL

### Sur une interface

    interface <type_interface> <numéro>
     ip access-group <numéro_ou_nom> in|out

- `in` : filtre le trafic **entrant** sur l'interface.
- `out` : filtre le trafic **sortant** de l'interface.

### Retirer d'une interface

    interface <type_interface> <numéro>
     no ip access-group <numéro_ou_nom> in|out

### Sur les lignes VTY (SSH / Telnet)

    line vty 0 4
     access-class <numéro_ou_nom> in

Restreint les connexions à distance selon l'ACL spécifiée.

### Retirer des lignes VTY

    line vty 0 4
     no access-class <numéro_ou_nom> in

---

## Vérification

    show access-lists

Affiche toutes les ACL et le compteur de correspondances (`matches`) par règle.

    show access-list <numéro_ou_nom>

Affiche une ACL spécifique.

    show ip interface <type_interface> <numéro>

Vérifie quelle ACL est appliquée en entrée (`Inbound access list`) et en sortie (`Outgoing access list`) sur une interface.
