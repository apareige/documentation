# Haute disponibilité d'un service web

> Documentation de mise en œuvre du TP B2-Act2-TP1 (contexte GSB) — cluster actif/passif Corosync + Pacemaker avec réplication MariaDB multi-maître.
> **Commandes remises à jour** pour Debian 12/13 (Corosync 3.x, Pacemaker 2.x, crmsh 4.x, MariaDB 10.11/11.x). Le support CERTA d'origine date de 2016 : une partie de sa syntaxe ne fonctionne plus — voir la section 9.
> Version 1.1 — corrections de vérification appliquées (voir §11 pour les sources).

## Sommaire

1. [Contexte et architecture](#1-contexte-et-architecture)
2. [Prérequis communs aux deux nœuds](#2-prérequis-communs-aux-deux-nœuds)
3. [Activité 1 — Corosync et Pacemaker](#3-activité-1--corosync-et-pacemaker)
4. [Activité 2 — Ressources IPFailover et serviceWeb](#4-activité-2--ressources-ipfailover-et-serviceweb)
5. [Activité 3 — Réplication MariaDB maître-esclave](#5-activité-3--réplication-mariadb-maître-esclave)
6. [Activité 4 — Multi-maître et intégration au cluster](#6-activité-4--multi-maître-et-intégration-au-cluster)
7. [Tests de validation](#7-tests-de-validation)
8. [Dépannage](#8-dépannage)
9. [Tableau des corrections 2016 vers aujourd'hui](#9-tableau-des-corrections-2016-vers-aujourdhui)
10. [Aide-mémoire des commandes](#10-aide-mémoire-des-commandes)
11. [Sources de vérification](#11-sources-de-vérification)

---

## 1. Contexte et architecture

| Élément | Valeur |
|---|---|
| Nœud maître | `serv1` — `<ip_serv1>` |
| Nœud esclave | `serv2` — `<ip_serv2>` |
| Adresse IP virtuelle (VIP) | `<vip>` / `<cidr>` |
| Services en HA | Apache2 (actif/passif), MariaDB (actif/actif cloné) |
| Outils | Corosync 3 (couche cluster), Pacemaker 2 (gestionnaire de ressources), crmsh (CLI) |

Principe : Corosync détecte l'état des nœuds via des battements de cœur (heartbeat), Pacemaker démarre/arrête les ressources. En actif/passif, le service est **installé sur les deux nœuds** mais **démarré uniquement sur le nœud actif**.

---

## 2. Prérequis communs aux deux nœuds

### 2.1 Nom d'hôte

```bash
hostnamectl set-hostname <nom_hote>
```
Définit le nom d'hôte de façon persistante. Ce nom doit correspondre **exactement** au champ `name:` de `corosync.conf`.

### 2.2 Résolution de noms

```bash
printf '%s\n' "<ip_serv1> serv1" "<ip_serv2> serv2" | tee -a /etc/hosts
```
Ajoute les deux nœuds dans `/etc/hosts`. Indispensable : le cluster ne doit pas dépendre d'un DNS externe pour se former.

### 2.3 Identifier le nom réel de l'interface réseau

```bash
ip -br link
```
Affiche les interfaces en format court. **`eth0` n'existe plus par défaut** : Debian utilise les noms prédictibles (`ens18`, `enp0s3`, `enp1s0`…). Ce nom sera réutilisé dans le paramètre `nic=` de la VIP.

### 2.4 Vérifier l'application avant clustering

```bash
curl -I http://<ip_serv1>/<chemin_appli>/
```
Vérifie que l'application de gestion de frais répond en HTTP avant d'ajouter la moindre couche HA (réponse `HTTP/1.1 200 OK` attendue).

### 2.5 Ouvrir les ports du cluster (si pare-feu actif)

```bash
ufw allow from <ip_autre_noeud> to any port 5405 proto udp
```
Corosync 3 (transport `knet`) échange sur **UDP/5405** entre les nœuds. Sans cette règle, le cluster ne se forme pas et chaque nœud se croit seul.

---

## 3. Activité 1 — Corosync et Pacemaker

### 3.1 Installation

```bash
apt update
apt install -y corosync pacemaker pacemaker-cli-utils crmsh fence-agents
```
`pacemaker-cli-utils` fournit `crm_mon`, `crm_verify`, `crm_resource` ; `crmsh` fournit la commande `crm`. `fence-agents` sert pour le STONITH en production. (Alternative : `pcs`, mais le TP CERTA est écrit pour `crm`.)

### 3.2 Désactiver le démarrage automatique des services gérés par le cluster

```bash
systemctl disable --now apache2
systemctl disable --now mariadb
```
⚠ **Étape critique, absente du support d'origine.** Tout service confié à Pacemaker doit être retiré du démarrage systemd, sinon les deux se disputent le contrôle : Apache tournerait aussi sur le nœud passif, et MariaDB serait démarrée avant que le cluster ne la sonde. C'est le cluster, et lui seul, qui démarre ces services.

> MariaDB sera configurée en clone actif/actif (§6.4), donc elle tournera bien sur les deux nœuds — mais démarrée **par Pacemaker**, pas par systemd.

### 3.3 Génération de la clé d'authentification

```bash
corosync-keygen
```
Génère `/etc/corosync/authkey` (256 octets, droits `0400`), qui authentifie et chiffre les échanges entre nœuds.
**Correction 2016 :** il n'y a plus à « taper au clavier pour générer de l'entropie » — depuis Corosync 2.x la clé est lue sur `/dev/urandom`, c'est instantané.

### 3.4 Copie de la clé sur l'autre nœud

```bash
scp /etc/corosync/authkey root@<ip_serv2>:/etc/corosync/authkey
ssh root@<ip_serv2> "chown root:root /etc/corosync/authkey && chmod 400 /etc/corosync/authkey"
```
La clé doit être **identique** sur tous les nœuds. Si `serv2` est issu d'un clonage de `serv1`, elle est déjà présente et cette étape est inutile.

### 3.5 Sauvegarde de la configuration d'origine

```bash
cp -a /etc/corosync/corosync.conf /etc/corosync/corosync.conf.ori
```
On **copie** au lieu de déplacer (le support utilise `mv`) : en cas de problème, le fichier de référence commenté reste disponible.

### 3.6 Fichier `/etc/corosync/corosync.conf` — version à jour

```ini
totem {
    version: 2
    cluster_name: <nom_cluster>
    transport: knet
    crypto_cipher: aes256
    crypto_hash: sha256
}

logging {
    to_logfile: yes
    logfile: /var/log/corosync/corosync.log
    to_syslog: yes
    timestamp: on
}

quorum {
    provider: corosync_votequorum
    two_node: 1
}

nodelist {
    node {
        name: serv1
        nodeid: 1
        ring0_addr: <ip_serv1>
    }
    node {
        name: serv2
        nodeid: 2
        ring0_addr: <ip_serv2>
    }
}
```

Différences par rapport au document CERTA, et pourquoi :

| Directive du support | Statut | Correction appliquée |
|---|---|---|
| `two_nodes: 1` | **faute de frappe** — directive inexistante, ignorée en silence | `two_node: 1` |
| `expected_votes: 2` | redondant | supprimé : déduit du `nodelist` |
| `clear_node_high_bit: yes` | obsolète | supprimé : les `nodeid` sont explicites |
| `crypto_hash: sha1` | algorithme faible | `sha256` |
| bloc `service { ver: 0 name: pacemaker }` | **supprimé en Pacemaker 2** | Pacemaker est une unité systemd indépendante |
| pas de `transport` | — | `transport: knet` (défaut Corosync 3, remplace le multicast UDP) |

`two_node: 1` active automatiquement `wait_for_all` : le cluster attend que les deux nœuds se soient vus une première fois avant d'accorder le quorum, puis un seul nœud suffit. C'est ce qui remplace proprement l'ancien `no-quorum-policy=ignore`.

### 3.7 Vérification syntaxique avant démarrage

```bash
corosync -t
```
Teste le fichier de configuration et sort sans démarrer le démon. À faire systématiquement avant un `systemctl restart`.

### 3.8 Démarrage des services

```bash
systemctl enable --now corosync pacemaker
systemctl status corosync pacemaker --no-pager
```
Active et démarre les deux unités. **`service corosync status` et `/etc/init.d/corosync status` du support sont à remplacer par `systemctl`** : les scripts SysV ne sont plus que des wrappers générés.

### 3.9 Clonage de la VM et préparation de `serv2`

```bash
hostnamectl set-hostname serv2
rm -f /etc/machine-id && systemd-machine-id-setup
rm -f /etc/ssh/ssh_host_* && dpkg-reconfigure openssh-server
```
⚠ **À exécuter uniquement sur le clone, jamais sur `serv1`.** Deux VM clonées partagent le même `machine-id` et les mêmes clés SSH : cela casse le DHCP, journald et l'authentification SSH. Point totalement absent du support de 2016.

Puis adapter l'adresse IP :

```bash
editor /etc/network/interfaces
systemctl restart networking
```
Adresse `<ip_serv2>` sur le clone. Sur une Debian sans `ifupdown`, la configuration se fait dans `/etc/systemd/network/` ou via `nmcli`.

Si le clone a conservé un état de cluster incohérent :

```bash
systemctl stop pacemaker corosync
rm -f /var/lib/pacemaker/cib/cib*
systemctl start corosync pacemaker
```
⚠ **Destructif** : efface la configuration du cluster locale. À ne faire que sur un nœud qui n'a pas encore rejoint le cluster ; la CIB sera resynchronisée depuis l'autre nœud.

### 3.10 Vérification du cluster sur les deux nœuds

```bash
corosync-cfgtool -s
```
État des liens (`LINK ID 0 ... status: connected`). La sortie de Corosync 3 diffère de celle du support (plus de « RING ID / ring 0 active with no faults »).

```bash
corosync-cfgtool -n
corosync-quorumtool -s
```
Liste des nœuds vus par la couche cluster et état du quorum (`Quorate: Yes`, `Flags: 2Node WaitForAll`).

```bash
crm status
```
Vue Pacemaker : nombre de nœuds, nœud DC, ressources. `crm_mon` donne la même chose en temps réel (sortie par `Ctrl+C`), `crm_mon -1` en une passe.

### 3.11 Désactivation de STONITH

```bash
crm configure property stonith-enabled=false
crm_verify -L -V
```
Sans agent de fencing déclaré, Pacemaker refuse de démarrer les ressources et `crm_verify` remonte des erreurs. On désactive **pour le labo uniquement**.
⚠ En production, on configure un vrai agent (`fence_virsh`, `fence_vmware_soap`, `fence_ipmilan` ou SBD) : sans fencing, un split-brain corrompt les données partagées.

### 3.12 Quorum sur deux nœuds

```bash
crm configure property no-quorum-policy=stop
```
Valeur par défaut, à laisser telle quelle : avec `two_node: 1` dans `corosync.conf`, le quorum est déjà géré correctement.

**Correction 2016 :** `no-quorum-policy="ignore"` reste une valeur acceptée en Pacemaker 2.x et 3.x, mais elle est déconseillée ici. Elle désactive globalement la politique de quorum, alors que `two_node: 1` traite précisément le cas des deux nœuds : le quorum est fixé artificiellement à 1, et `wait_for_all` (activé automatiquement par `two_node`) évite la course au démarrage où chaque nœud se croirait seul. On garde donc `stop`.

### 3.13 Consultation de la configuration générée

```bash
crm configure show
```
Affiche la CIB en syntaxe crmsh, bien plus lisible que le XML.

```bash
cibadmin -Q | head -n 40
```
Affiche la CIB brute au format XML. **Ne jamais éditer `/var/lib/pacemaker/cib/cib.xml` à la main** — utiliser `crm configure edit`.

---

## 4. Activité 2 — Ressources IPFailover et serviceWeb

### 4.1 Création de la ressource d'adresse IP virtuelle

```bash
crm configure primitive IPFailover ocf:heartbeat:IPaddr2 \
  params ip=<vip> cidr_netmask=<cidr> nic=<interface> iflabel=VIP \
  op monitor interval=10s timeout=20s \
  op start interval=0s timeout=30s \
  op stop interval=0s timeout=40s
```
Crée la VIP demandée en Q1 : test de vie toutes les 10 s, timeout de démarrage 30 s, d'arrêt 40 s.
`ocf` = classe, `heartbeat` = fournisseur, `IPaddr2` = script appelé. `iflabel=VIP` crée un alias visible d'interface. `nic=eth0` du support doit être remplacé par le nom réel relevé en 2.3.

⚠ Le label produit l'alias `<interface>:VIP`, soumis à la limite noyau `IFNAMSIZ` de 15 caractères. `ens18:VIP` (9) ou `enp0s31f6:VIP` (13) passent ; une interface longue plus un label long font échouer le démarrage de la ressource avec une erreur peu explicite. En cas de doute, raccourcir le label ou retirer `iflabel` (la VIP reste visible avec `ip addr show`).

```bash
crm ra info ocf:heartbeat:IPaddr2
```
Liste tous les paramètres et timeouts par défaut de l'agent — à consulter avant d'inventer des valeurs.

### 4.2 Vérifier où la ressource a démarré

```bash
crm status
ip -c addr show dev <interface>
```
Pacemaker place la ressource sur un nœud **arbitrairement** tant qu'aucune contrainte n'existe. La VIP apparaît comme adresse `secondary` sur l'interface.
**Correction 2016 :** `ifconfig` n'est plus installé par défaut (`net-tools`) — utiliser `ip addr`.

### 4.3 Définir une préférence de nœud

```bash
crm resource move IPFailover <noeud>
```
Migre la ressource et fait écrire par Pacemaker une contrainte de localisation `cli-prefer-IPFailover`.

```bash
crm resource clear IPFailover
```
Supprime cette contrainte une fois la migration constatée.

**Correction 2016 :** `clear` est le nom actuel de la commande en crmsh 4 ; `unmove` reste accepté comme alias, donc les deux fonctionnent. Ce qui compte, c'est de ne pas oublier l'étape : sans elle, la contrainte reste **permanente** et le comportement du cluster devient difficile à expliquer en soutenance.

### 4.4 Empêcher le retour automatique (auto failback)

```bash
crm configure rsc_defaults resource-stickiness=100
```
La ressource reste sur le nœud où elle tourne même si le nœud préféré revient en ligne ; le retour devient manuel.
**Correction 2016 :** `crm configure property default-resource-stickiness=100` n'existe plus en Pacemaker 2 — c'est `rsc_defaults resource-stickiness`.

### 4.5 Tester l'accès par la VIP

```bash
curl -I http://<vip>/<chemin_appli>/
```
Valide la Q4 : l'application répond bien sur l'adresse virtuelle.

### 4.6 Service impacté par le changement d'adresse (Q5)

```bash
grep -R "ServerName" /etc/apache2/sites-enabled/
```
C'est le **DNS** qui est impacté : l'enregistrement A du FQDN de l'application doit pointer sur `<vip>`, plus sur l'IP réelle d'un nœud. Corollaire côté Apache : `ServerName` du vhost, et surtout le **certificat TLS** (CN/SAN) doivent correspondre à ce FQDN, sinon HTTPS casse à la bascule.

### 4.7 Ressource du service Web

```bash
crm configure primitive serviceWeb systemd:apache2 \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=60s \
  op stop interval=0s timeout=60s
```
**Correction majeure :** le support utilise `lsb:apache2`. La classe LSB s'appuie sur `/etc/init.d/apache2`, qui n'est plus qu'un wrapper systemd sur Debian moderne : le monitoring est peu fiable. On utilise la classe `systemd:`.

Variante plus fine, avec supervision HTTP réelle :

```bash
a2enmod status
crm configure primitive serviceWeb ocf:heartbeat:apache \
  params configfile=/etc/apache2/apache2.conf \
         statusurl="http://127.0.0.1/server-status" \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=60s \
  op stop interval=0s timeout=60s
```
L'agent OCF interroge `/server-status` : il détecte un Apache « lancé mais qui ne répond plus », ce que `systemd:` ne voit pas. Nécessite `mod_status` activé et accessible depuis `127.0.0.1`.

### 4.8 Vérifier que le service ne tourne que sur un nœud (Q8)

```bash
ss -lntp | grep ':80'
```
Sur le nœud actif, Apache écoute ; sur le passif, aucune ligne.
**Correction 2016 :** `netstat` est remplacé par `ss` (paquet `net-tools` non installé). `nmap localhost -p 80` reste valable mais `ss` suffit et est natif.

### 4.9 Grouper les ressources (Q9)

```bash
crm configure group servweb IPFailover serviceWeb meta migration-threshold=5
```
Par défaut Pacemaker **répartit** les ressources entre nœuds : la VIP et Apache se retrouvent séparés. Le groupe force la colocation et un ordre de démarrage (VIP puis Apache, arrêt en sens inverse). `migration-threshold=5` : au bout de 5 échecs, le groupe migre définitivement.

```bash
crm configure rsc_defaults failure-timeout=600s
```
Complément utile : les compteurs d'échec s'effacent seuls au bout de 10 minutes, ce qui évite de devoir faire un `cleanup` manuel après chaque test.

⚠ `failure-timeout` est un **méta-attribut de ressource**, pas une propriété de cluster. Passé en `crm configure property`, Pacemaker l'enregistre sans broncher (il tolère les noms d'options personnalisés) mais **ne l'applique jamais** — panne silencieuse difficile à diagnostiquer. Il se met soit dans `rsc_defaults`, soit directement sur la ressource avec `meta failure-timeout=600s`.

---

## 5. Activité 3 — Réplication MariaDB maître-esclave

### 5.1 Installation et point de vocabulaire

```bash
apt install -y mariadb-server
systemctl status mariadb --no-pager
```
Sur Debian, le paquet est `mariadb-server` et l'unité `mariadb.service` (`mysql.service` n'est qu'un alias). Les binaires clients sont `mariadb`, `mariadb-dump`, `mariadb-admin` — `mysql` et `mysqldump` ne sont plus que des liens de compatibilité.

### 5.2 Fichier `/etc/mysql/mariadb.conf.d/50-server.cnf` — nœud maître

```ini
[mysqld]
bind-address               = 0.0.0.0
log_error                  = /var/log/mysql/error.log
server_id                  = 100
log_bin                    = /var/log/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 864000
max_binlog_size            = 100M
binlog_do_db               = <nom_base>
```
`server_id` doit être unique dans la chaîne de réplication. `log_bin` active le journal binaire lu par l'esclave. `binlog_do_db` restreint la réplication à la base voulue.

**Corrections 2016 :** `expire_logs_days = 10` est déprécié → `binlog_expire_logs_seconds = 864000` (= 10 jours). On **définit** `bind-address = 0.0.0.0` au lieu de commenter la ligne. `binlog_format = ROW` est ajouté, et ce n'est pas cosmétique : en statement-based, `binlog_do_db` et `replicate_do_db` filtrent sur la base **par défaut** (celle du dernier `USE`) et non sur la table réellement écrite. Une requête `INSERT INTO autrebase.table` lancée depuis `USE <nom_base>` serait donc répliquée à tort, et l'inverse serait ignoré. En ROW, le filtrage porte sur la table réelle.

### 5.3 Même fichier — nœud esclave

```ini
[mysqld]
bind-address       = 0.0.0.0
log_error          = /var/log/mysql/error.log
server_id          = 104
log_bin            = /var/log/mysql/mysql-bin.log
relay_log          = /var/log/mysql/mysql-relay-bin.log
log_slave_updates  = ON
replicate_do_db    = <nom_base>
master_retry_count = 100000
```
`log_slave_updates` est ajouté dès maintenant : il sera indispensable en activité 4.

**Correction du support 2016 :** `master-retry-count` **reste une option serveur valide** et se met bien dans le `.cnf`. C'est sa *description* qui est fausse dans le support : ce n'est pas « se reconnecter toutes les 20 secondes », c'est le **nombre de tentatives** de connexion avant abandon définitif. L'intervalle entre deux tentatives, lui, est fixé par `MASTER_CONNECT_RETRY` (60 s par défaut).

⚠ Ne pas reprendre la valeur `20` du support : l'esclave abandonnerait après 20 essais, soit une vingtaine de minutes d'indisponibilité du maître. Les valeurs par défaut sont 86400 jusqu'à MariaDB 10.5 et 100000 à partir de 10.6 — on les garde.

### 5.4 Redémarrage

```bash
systemctl restart mariadb
journalctl -u mariadb -n 30 --no-pager
```
Toujours relire les logs après redémarrage : une directive inconnue empêche MariaDB de démarrer (`unknown variable`).

### 5.5 Création du compte de réplication (sur le maître)

```sql
CREATE USER '<user_repl>'@'<ip_serv2>' IDENTIFIED BY '<mdp_repl>';
GRANT REPLICATION SLAVE ON *.* TO '<user_repl>'@'<ip_serv2>';
FLUSH PRIVILEGES;
```
**Correction 2016 :** MariaDB accepte encore `GRANT ... IDENTIFIED BY '<mdp>'`, mais MySQL 8 l'a supprimée. La forme `CREATE USER` **puis** `GRANT` fonctionne sur les deux : c'est celle à retenir. Le privilège `REPLICATION SLAVE` suffit, ne pas donner `ALL`.

Connexion à la console SQL :

```bash
mariadb -u root -p
```
Sur Debian, `root` utilise l'authentification `unix_socket` : `sudo mariadb` fonctionne sans mot de passe.

### 5.6 Synchronisation initiale des données

Méthode recommandée aujourd'hui, sans verrou global prolongé :

```bash
mariadb-dump --single-transaction --master-data=2 --databases <nom_base> > /root/<fichier_dump>.sql
```
`--single-transaction` fige une vue cohérente (InnoDB) sans bloquer les écritures ; `--master-data=2` ajoute en tête du dump, sous forme de commentaire, le `CHANGE MASTER TO` avec le fichier et la position exacts.

⚠ `--source-data` est le nom **MySQL 8.0.26+** de cette option. MariaDB ne le connaît pas et sort `unknown option`. Sur MariaDB, c'est bien `--master-data`.

**Corrections 2016 :** `mysqldump` → `mariadb-dump` ; ne jamais passer le mot de passe en clair via `-p<motDePasse>` (visible dans `ps`), utiliser `-p` seul ou un fichier `--defaults-extra-file`.

Méthode « historique » du support, si l'on veut respecter le TP à la lettre :

```sql
FLUSH TABLES WITH READ LOCK;
SHOW BINLOG STATUS;
```
Bloque toutes les écritures et affiche `File` et `Position` à noter. `SHOW MASTER STATUS` reste accepté ; `SHOW BINLOG STATUS` est la forme actuelle côté MariaDB depuis 10.5.2 (`SHOW BINARY LOG STATUS` côté MySQL 8.2+).
⚠ Tant que `UNLOCK TABLES;` n'est pas saisi, l'application est en lecture seule — ne pas laisser la session ouverte.

Restauration sur l'esclave :

```bash
mariadb < /root/<fichier_dump>.sql
```
Les bases doivent être **strictement identiques** avant de lancer la réplication, sinon erreur `Duplicate entry ... Error_code: 1062`.

### 5.7 Configuration de l'esclave

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='<ip_serv1>',
  MASTER_USER='<user_repl>',
  MASTER_PASSWORD='<mdp_repl>',
  MASTER_LOG_FILE='<fichier_binlog>',
  MASTER_LOG_POS=<position>,
  MASTER_CONNECT_RETRY=20;
START SLAVE;
```
Indique au nœud esclave depuis quel serveur, quel fichier et quelle position répliquer. `MASTER_CONNECT_RETRY=20` fixe le délai en secondes entre deux tentatives de reconnexion.

⚠ **Ne pas ajouter `MASTER_RETRY_COUNT` ici.** Cette option de `CHANGE MASTER TO` n'existe qu'à partir de MariaDB 12.0.1 (et côté MySQL). Sur Debian 12 (MariaDB 10.11) et Debian 13 (11.8), elle provoque une erreur de syntaxe qui fait échouer toute la commande. Le nombre de tentatives se règle dans le `.cnf` (voir §5.3).

Variante GTID (recommandée en MariaDB 10.x/11.x, plus robuste) :

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='<ip_serv1>',
  MASTER_USER='<user_repl>',
  MASTER_PASSWORD='<mdp_repl>',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
```
Plus besoin de relever fichier/position à la main : le GTID suit tout seul le point de reprise, y compris après une bascule. C'est l'approche moderne, absente du support de 2016.

Puis, sur le maître, si l'on avait verrouillé :

```sql
UNLOCK TABLES;
```
Libère les écritures. À faire **immédiatement** après avoir configuré l'esclave.

### 5.8 Contrôle de l'état de la réplication

```sql
SHOW SLAVE STATUS\G
```
Trois champs à vérifier : `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes`, `Seconds_Behind_Master: 0`. En cas d'erreur, lire `Last_IO_Error` et `Last_SQL_Error`.
**Note de syntaxe :** MariaDB 10.5+ accepte les alias `SHOW REPLICA STATUS`, `START REPLICA`, `STOP REPLICA`. Sous **MySQL 8.0.23+**, ces formes sont **obligatoires** : `CHANGE REPLICATION SOURCE TO SOURCE_HOST=..., SOURCE_USER=..., SOURCE_LOG_FILE=..., SOURCE_LOG_POS=...`.

---

## 6. Activité 4 — Multi-maître et intégration au cluster

### 6.1 Compléter les deux fichiers de configuration

Ajouter sur **chaque** nœud ce qui lui manque pour jouer les deux rôles :

```ini
[mysqld]
server_id                = <id_unique>
log_bin                  = /var/log/mysql/mysql-bin.log
relay_log                = /var/log/mysql/mysql-relay-bin.log
log_slave_updates        = ON
binlog_format            = ROW
binlog_do_db             = <nom_base>
replicate_do_db          = <nom_base>
auto_increment_increment = 2
auto_increment_offset    = <1_ou_2>
```
`log_slave_updates` est **obligatoire** : sans lui, un nœud ne journalise pas dans son binlog les écritures reçues par réplication, et il ne peut pas détecter qu'un événement qu'il a déjà exécuté lui revient (boucle infinie). `auto_increment_increment/offset` évitent les collisions de clés auto-incrémentées (pas de 2, départ à 1 sur un nœud et 2 sur l'autre).

```bash
systemctl restart mariadb
```
Redémarrage nécessaire sur les deux nœuds après modification.

### 6.2 Créer le compte de réplication dans l'autre sens

```sql
CREATE USER '<user_repl>'@'<ip_serv1>' IDENTIFIED BY '<mdp_repl>';
GRANT REPLICATION SLAVE ON *.* TO '<user_repl>'@'<ip_serv1>';
FLUSH PRIVILEGES;
```
À exécuter sur `serv2`, qui devient à son tour maître de `serv1`.

### 6.3 Transformer `serv1` en esclave de `serv2`

```sql
SHOW BINLOG STATUS;
```
Sur `serv2` : relever `File` et `Position`.

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='<ip_serv2>',
  MASTER_USER='<user_repl>',
  MASTER_PASSWORD='<mdp_repl>',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
SHOW SLAVE STATUS\G
```
À exécuter sur `serv1`. On obtient deux réplications maître-esclave croisées : chaque nœud peut reprendre l'activité avec des données à jour, sans procédure manuelle de bascule.

### 6.4 Ressource MariaDB en mode actif/actif

Solution simple et fiable pour le TP :

```bash
crm configure primitive serviceMySQL systemd:mariadb \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=120s \
  op stop interval=0s timeout=120s
crm configure clone cServiceMySQL serviceMySQL meta interleave=true
```
Le clone de type `anonymous` démarre la ressource **sur les deux nœuds** : il n'y a pas de bascule de MariaDB, chaque nœud a déjà sa base à jour.

Variante avec l'agent OCF (plus proche du support, plus capricieuse) :

```bash
crm configure primitive serviceMySQL ocf:heartbeat:mysql \
  params binary=/usr/sbin/mariadbd \
         config=/etc/mysql/mariadb.cnf \
         datadir=/var/lib/mysql \
         pid=/run/mysqld/mysqld.pid \
         socket=/run/mysqld/mysqld.sock \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=120s \
  op stop interval=0s timeout=120s
```
**Corrections 2016 :** le support écrit `ocf:Corosync:mysql` — le fournisseur est **`heartbeat`**, pas `Corosync`. Le binaire est `mariadbd` depuis MariaDB 10.5 (et non `mysqld`). Le socket canonique est `/run/mysqld/mysqld.sock` (`/var/run` n'est qu'un lien symbolique).

### 6.5 Ordonner le démarrage

```bash
crm configure order o-mysql-avant-web Mandatory: cServiceMySQL servweb
```
Garantit que MariaDB est opérationnel avant qu'Apache ne démarre sur le nœud actif — sinon l'application affiche une erreur de connexion à la base pendant quelques secondes après chaque bascule.

```bash
crm configure show
crm status
```
Contrôle final : le groupe `servweb` sur un seul nœud, le `Clone Set: cServiceMySQL` démarré sur les deux.

---

## 7. Tests de validation

### 7.1 Mise en maintenance d'un nœud

```bash
crm node standby <noeud>
crm status
```
Le nœud n'héberge plus de ressources : le groupe bascule sur l'autre. Bascule **propre**, les services sont arrêtés puis redémarrés dans l'ordre.

```bash
crm node online <noeud>
```
Remet le nœud en service. Avec `resource-stickiness=100`, les ressources **ne reviennent pas** automatiquement.

### 7.2 Extinction brutale du nœud actif

```bash
systemctl poweroff
```
⚠ Test destructif pour les sessions en cours. Différence avec le standby : Corosync doit d'abord détecter la perte du nœud (délai de token), la bascule est donc plus lente et les connexions en cours sont coupées.

### 7.3 Panne du seul service Web

```bash
systemctl mask apache2
systemctl stop apache2
```
⚠ Empêche volontairement tout redémarrage d'Apache, ce qui force Pacemaker à échouer puis à migrer le groupe après `migration-threshold` échecs. Sans cela, Pacemaker relance le service et **aucune bascule n'a lieu** — c'est le piège du TP.

```bash
systemctl unmask apache2
crm resource cleanup servweb
```
**Retour arrière obligatoire** après le test : démasque le service et remet à zéro les compteurs d'erreur, sans quoi le nœud reste inéligible pour héberger les ressources.

### 7.4 Validation de la réplication

```sql
SELECT COUNT(*) FROM <nom_base>.<table>;
```
À exécuter sur les deux nœuds après saisie d'une fiche de frais depuis l'application : les compteurs doivent être identiques.

### 7.5 Tableau de recette (Q3 act.3 / Q4 act.4)

| Action à effectuer | Résultat attendu | Résultat obtenu | Statut |
|---|---|---|---|
| Ressources sur `serv1`, saisie d'une fiche de frais | Fiche visible sur les deux bases | | |
| `crm node standby serv1` | Groupe `servweb` démarré sur `serv2`, VIP répond | | |
| Saisie d'une seconde fiche via la VIP | Écriture acceptée sur `serv2` | | |
| `crm node online serv1` | `serv1` en ligne, réplication `Yes/Yes` dans les deux sens | | |
| Comparaison des deux bases | Nombre de fiches identique des deux côtés | | |

---

## 8. Dépannage

### 8.1 Logs du cluster

```bash
journalctl -u corosync -u pacemaker -f
```
Flux temps réel des deux démons. Complément : `/var/log/corosync/corosync.log` et `/var/log/pacemaker/pacemaker.log`.

```bash
crm_mon -rfn1
```
Vue détaillée en une passe : ressources inactives (`-r`), compteurs d'échec (`-f`), regroupées par nœud (`-n`). La commande la plus utile pour comprendre pourquoi une ressource ne démarre pas.

### 8.2 Compteurs d'échec

```bash
crm resource failcount <id_ressource> show <noeud>
crm resource cleanup <id_ressource>
```
Après un certain nombre d'erreurs, le nœud fautif ne peut plus héberger la ressource tant que les compteurs ne sont pas effacés.

### 8.3 Sauvegarde et restauration de la configuration

```bash
crm configure save /root/<fichier_sauvegarde>.conf
crm configure save xml /root/<fichier_sauvegarde>.xml
```
Export de la CIB. À faire **avant** toute manipulation risquée.

```bash
crm configure load replace /root/<fichier_sauvegarde>.conf
```
⚠ **Remplace intégralement** la configuration courante. Pour un ajout, utiliser `load update` à la place.

### 8.4 Erreur de réplication `Duplicate entry ... 1062`

```sql
SHOW SLAVE STATUS\G
```
Lire `Last_SQL_Error` pour identifier la table et la position en cause.

```sql
STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;
```
⚠ **Dangereux** : saute l'événement en erreur et crée une divergence silencieuse entre les bases. Acceptable en labo pour débloquer, jamais en production sans analyse.

⚠ Ce compteur saute un événement du log binaire, ce qui n'a de sens qu'en réplication par fichier/position. **Si l'esclave a été configuré avec `MASTER_USE_GTID=slave_pos` (§5.7), c'est le mauvais outil** : il faut faire avancer la position GTID.

```sql
STOP SLAVE;
SET GLOBAL gtid_slave_pos = '<domaine>-<serveur>-<sequence>';
CHANGE MASTER TO MASTER_USE_GTID=slave_pos;
START SLAVE;
```
Encore plus dangereux que le compteur : on déclare explicitement à l'esclave qu'il a traité une transaction qu'il n'a pas exécutée. La position courante se lit avec `SELECT @@global.gtid_slave_pos;` et celle du maître avec `SELECT @@global.gtid_binlog_pos;`. Dans les deux cas, la solution propre reste le redump du maître vers l'esclave (§5.6).

### 8.5 Gestion des journaux binaires

```sql
SHOW BINARY LOGS;
PURGE BINARY LOGS TO '<fichier_binlog>';
```
Le maître ignore combien d'esclaves le suivent : ses binlogs s'accumulent. `PURGE` supprime tout jusqu'au fichier indiqué, exclu — ne jamais purger le fichier en cours.
**Correction 2016 :** `SHOW MASTER LOGS` / `PURGE MASTER LOGS` sont remplacés par `SHOW BINARY LOGS` / `PURGE BINARY LOGS`.
⚠ Ne jamais supprimer ces fichiers avec `rm` : l'index `mysql-bin.index` se désynchronise et `PURGE` cesse de fonctionner.

---

## 9. Tableau des corrections 2016 vers aujourd'hui

| Support CERTA (2016) | Version actuelle | Motif |
|---|---|---|
| `service corosync status`, `/etc/init.d/corosync status` | `systemctl status corosync` | SysV remplacé par systemd |
| `two_nodes: 1` | `two_node: 1` | directive inexistante, ignorée en silence |
| `expected_votes: 2` | (supprimé) | déduit du `nodelist` |
| `clear_node_high_bit: yes` | (supprimé) | obsolète avec `nodeid` explicites |
| `service { ver: 0 name: pacemaker }` | (supprimé) + `systemctl enable pacemaker` | Pacemaker 2 n'est plus démarré par Corosync |
| `crypto_hash: sha1` | `crypto_hash: sha256` | SHA-1 obsolète |
| multicast implicite | `transport: knet` | défaut Corosync 3 |
| `no-quorum-policy=ignore` | `two_node: 1` + `no-quorum-policy=stop` | `ignore` toujours valide mais déconseillé : `two_node` couvre le cas proprement |
| `default-resource-stickiness=100` | `rsc_defaults resource-stickiness=100` | propriété supprimée en Pacemaker 2.0.0 |
| `crm resource unmove` | `crm resource clear` | `clear` est le nom actuel, `unmove` reste un alias |
| `lsb:apache2` | `systemd:apache2` ou `ocf:heartbeat:apache` | classe LSB non fiable sous systemd |
| `ocf:Corosync:mysql` | `ocf:heartbeat:mysql` ou `systemd:mariadb` | fournisseur erroné dans le support |
| `binary=mysqld` | `binary=/usr/sbin/mariadbd` | binaire renommé (MariaDB 10.5+) |
| `/var/run/mysqld/mysqld.sock` | `/run/mysqld/mysqld.sock` | `/var/run` n'est qu'un lien |
| `nic=eth0` | `nic=<ens18 / enp0s3>` | noms d'interfaces prédictibles |
| `ifconfig` | `ip -c addr show` | `net-tools` non installé |
| `netstat` | `ss -lntp` | idem |
| `corosync-keygen` + frappe clavier | `corosync-keygen` (instantané) | entropie lue sur `/dev/urandom` |
| `mysqldump`, `mysql` | `mariadb-dump`, `mariadb` | binaires renommés |
| `expire_logs_days = 10` | `binlog_expire_logs_seconds = 864000` | directive dépréciée |
| `master-retry-count = 20` | `master_retry_count = 100000` (dans le `.cnf`) | l'option est valide : c'est sa description qui était fausse — nombre de tentatives, pas un délai |
| `GRANT ... IDENTIFIED BY` | `CREATE USER` puis `GRANT` | supprimé en MySQL 8 ; encore accepté en MariaDB mais à éviter |
| `#bind-address = 127.0.0.1` | `bind-address = 0.0.0.0` | plus explicite et reproductible |
| `SHOW MASTER STATUS` | `SHOW BINLOG STATUS` (MariaDB 10.5.2+) | renommage, ancien nom conservé comme alias |
| `SHOW MASTER LOGS` / `PURGE MASTER LOGS` | `SHOW BINARY LOGS` / `PURGE BINARY LOGS` | renommage |
| `STOP/START SLAVE`, `SHOW SLAVE STATUS` | alias `REPLICA` (MariaDB 10.5+), **obligatoire** en MySQL 8.0.23+ | terminologie révisée |
| position binlog relevée à la main | `MASTER_USE_GTID=slave_pos` | GTID, reprise automatique |
| (absent) | `systemctl disable --now apache2 mariadb` | sinon systemd et Pacemaker se disputent les services |
| (absent) | `crm configure rsc_defaults failure-timeout=600s` | `failure-timeout` est un méta-attribut, ignoré s'il est posé en `property` |
| (absent) | `binlog_format = ROW` | `binlog_do_db` / `replicate_do_db` filtrent sur la base par défaut en statement-based |
| (absent) | régénération `/etc/machine-id` + clés SSH après clonage | conflits d'identité entre VM clonées |
| `stonith-enabled=false` | idem en labo, agent réel en production | sans fencing, risque de split-brain |

---

## 10. Aide-mémoire des commandes

```bash
crm status                                   # état du cluster, une passe
crm_mon -rfn                                 # temps réel, détaillé (Ctrl+C pour sortir)
corosync-quorumtool -s                       # état du quorum
crm configure show                           # configuration lisible
crm configure edit [<id_ressource>]          # édition dans vi, appliquée à la sauvegarde
crm configure verify                         # validation de la configuration
crm resource start|stop <id_ressource>       # démarrage / arrêt d'une ressource
crm resource move <id_ressource> <noeud>     # migration (crée une contrainte)
crm resource clear <id_ressource>            # suppression de la contrainte (alias : unmove)
crm resource cleanup <id_ressource>          # effacement des compteurs d'erreur
crm configure delete <id_ressource>          # suppression (stopper la ressource avant)
crm node standby|online <noeud>              # entrée / sortie de maintenance
crm ra list ocf heartbeat                    # agents OCF disponibles
crm ra info ocf:heartbeat:<agent>            # paramètres et timeouts par défaut
crm configure clone <nom_clone> <id_ressource>   # ressource en actif/actif
crm configure rsc_defaults <option>=<valeur>     # méta-attribut par défaut de toutes les ressources
crm configure save /root/<fichier>.conf      # sauvegarde de la configuration
crm configure load update /root/<fichier>.conf   # application incrémentale
```

```sql
SHOW BINLOG STATUS;                          -- fichier et position courants (maître)
SHOW SLAVE STATUS\G                          -- état complet de l'esclave
STOP SLAVE; START SLAVE;                     -- arrêt / relance du thread de réplication
SELECT @@global.gtid_slave_pos;              -- position GTID de l'esclave
SELECT @@global.gtid_binlog_pos;             -- position GTID du maître
SHOW BINARY LOGS;                            -- liste des journaux binaires
PURGE BINARY LOGS TO '<fichier_binlog>';     -- purge jusqu'au fichier indiqué (exclu)
```

---

## 11. Sources de vérification

Les corrections apportées au support CERTA 2016 ont été vérifiées dans :

- **Corosync 3** : manpages `votequorum(5)` et `corosync(8)`, Debian bookworm — comportement de `two_node` / `wait_for_all`, option `-t`
- **Pacemaker 2.1** : *Pacemaker Explained*, chapitre « Resource Operations » (`failure-timeout` comme méta-attribut) et `pacemaker-schedulerd(7)` (valeurs acceptées de `no-quorum-policy`)
- **Pacemaker 2.0.0** release notes — suppression des alias `default-resource-stickiness`, `is-managed-default`, `default-action-timeout`
- **crmsh** : *Quick Comparison of pcs and crm shell*, documentation ClusterLabs 2.1
- **MariaDB KB** : `CHANGE MASTER TO` (disponibilité de `MASTER_RETRY_COUNT` à partir de 12.0.1), `SHOW BINLOG STATUS` (10.5.2), options `mariadbd` (`--master-retry-count`), options `mariadb-dump` (`--master-data`)
- **MySQL 8.0 Reference Manual** : `mysqldump` (`--source-data`, 8.0.26+)
