# Documentation FOG - Installation et Utilisation

> Guide complet pour l'installation et l'utilisation du serveur FOG Project pour le déploiement d'images système

## Table des matières

- [Matériel requis](#matériel-requis)
- [Installation et commande](#installation-et-commande)
- [Ajout du FOG au DHCP](#ajout-du-fog-au-dhcp)
- [Création d'une image](#création-dune-image)
  - [Création d'une image sur le serveur](#création-dune-image-sur-le-serveur)
  - [Création d'image pour Windows 10/11 sans fichier de réponse](#création-dimage-pour-windows-1011-sans-fichier-de-réponse)
  - [Création d'image pour Windows 10/11 avec fichier de réponse](#création-dimage-pour-windows-1011-avec-fichier-de-réponse)

---

## Matériel requis

### Processeur

- Processeur x86/x64 moderne avec prise en charge de la virtualisation (si nécessaire)

### Mémoire RAM

- **Minimum** : 2 Go (pour un petit environnement avec quelques machines)
- **Recommandé** : 4 Go ou plus (pour gérer un nombre important de machines)

### Espace disque

- **Minimum** : 100 Go (pour les images système et les fichiers associés)
- **Recommandé** : 500 Go ou plus (en fonction du nombre d'images et de leur taille)

> [!IMPORTANT]
> Les images sont stockées dans le répertoire `/images`, qui appartient à la partition système. Il faut donc dimensionner en conséquence cette partition ou choisir dans l'installation de Debian « tout dans la même partition ».

### Carte réseau

- Interface réseau rapide (Gigabit recommandé)

---

## Installation et commande

> [!NOTE]
> Pensez à mettre à jour le système et à définir l'IP du FOG à l'avance en la changeant dans le fichier de configuration (Debian : `/etc/network/interfaces`).

### Étapes d'installation

1. **Installer git**
   
   ```bash
   sudo apt install git
   ```

2. **Aller dans le répertoire tmp**
   
   ```bash
   cd /tmp
   ```

3. **Télécharger FOG**
   
   ```bash
   git clone https://github.com/FOGProject/fogproject.git
   ```

4. **Aller dans le répertoire de FOG**
   
   ```bash
   cd /fogproject/bin
   ```

5. **Lancer l'installeur de FOG**
   
   ```bash
   sudo ./installfog.sh
   ```

6. **Suivre les instructions de l'installeur**

> [!WARNING]
> Activer HTTPS peut causer des problèmes !

> 

### Message de fin d'installation

À la fin de l'installation, un message de ce type devrait apparaître :

```
* Setup complete

You can now login to the FOG Management Portal using
the information listed below. The login information
is only if this is the first install.

This can be done by opening a web browser and going to:
http://x.x.x.x/fog/management

Default User Information
Username: fog
Password: password
```

Désormais, le FOG est installé mais n'est pas encore prêt.

---

## Ajout du FOG au DHCP

Le serveur DHCP va devoir être configuré pour permettre le démarrage des machines en PXE (via le réseau) soit en mode Legacy (obsolète) ou EFI.

Consultez la documentation officielle : [DHCP Server Settings - FOG Project Documentation](https://docs.fogproject.org/en/latest/installation/server/dhcp.html)

---

## Création d'une image

> [!IMPORTANT]
> Créer une image prend du temps et peut être fastidieux, cela peut prendre 1 h hors temps machine.

### Recommandations générales

Pour créer l'image, il faut qu'elle soit la plus neutre possible :

- Télécharger la dernière version de Windows
- Installer Windows normalement et créer un compte local administrateur
- Éviter le plus possible les mises à jour de Windows
- Ne rien installer en provenance du Microsoft Store pour éviter les erreurs au moment du « Sysprep » (généralisation de l'image)

---

## Création d'une image sur le serveur

### 1. Créer l'image dans l'interface FOG

Sur un autre PC, sur la page de gestion du FOG :

- Aller dans **Image Management**
- Créer une nouvelle image pour Windows

> [!NOTE]
> Dans la liste des Template, Windows 11 n'apparaît pas, c'est normal. Gardez Windows 10.

### 2. Installer FOG Client sur la machine source

1. Ouvrir le navigateur et aller sur la page de votre FOG : `http://x.x.x.x/fog/management`
2. Sans vous connecter, vous devrez trouver **« FOG Client »** en bas de la page
3. Téléchargez-le puis exécutez-le
4. Suivez les instructions jusqu'à la page de configuration :
   - Remplacer `fog.company.com` par l'IP de votre FOG
   - Cliquer sur **Next** et l'installation est finie

### 3. Approuver la machine dans FOG

Désormais il faut approuver la machine dans l'interface FOG. L'installation de FOG Client est finie.

---

## Création d'image pour Windows 10/11 sans fichier de réponse

> [!WARNING]
> Pour la suite, toutes les applications seront à ouvrir en administrateur.

### 4. Exécuter Sysprep

1. Aller chercher le fichier sysprep dans `C:\Windows\System32\Sysprep`
2. Exécuter `sysprep.exe`
3. Choisir **Généraliser** et **Éteindre**
4. Avec beaucoup de chance, il n'y aura aucune erreur

### 5. En cas d'erreur

Normalement le chemin du fichier d'erreur s'affiche. Éditez le fichier de log et trouvez la ligne qui correspond à l'erreur.

Les commandes ci-dessous devraient en résoudre pas mal :

```powershell
Get-AppxPackage Microsoft.OneDriveSync | Remove-AppxPackage
Get-AppxPackage MicrosoftWindows.Client.WebExperience | Remove-AppxPackage
```

> [!TIP]
> Remplacez par exemple `Microsoft.OneDriveSync` par ce qui cause l'erreur et relancez sysprep jusqu'à réussite.

### 6. Créer la tâche de capture

Normalement l'ordinateur s'éteint. Dans FOG, créez la tâche de capture.

### 7. Lancer la capture

1. Redémarrer l'ordinateur en mode **boot PXE**
2. Ne plus toucher à l'ordinateur, FOG fait le reste
3. À la fin de la capture, l'ordinateur va redémarrer

---

## Création d'image pour Windows 10/11 avec fichier de réponse

### 1. Exécuter Sysprep avec le fichier de réponse

1. Aller chercher le fichier sysprep dans `C:\Windows\System32\Sysprep`
2. **Ajouter le fichier `unattend.xml`** dans ce répertoire
3. Exécuter `sysprep.exe`
4. Choisir **Généraliser** et **Éteindre**
5. Avec beaucoup de chance, il n'y aura aucune erreur

### 2. En cas d'erreur

Normalement le chemin du fichier d'erreur s'affiche. Ouvrez-le et trouvez la ligne qui correspond à l'erreur.

Les commandes ci-dessous devraient en résoudre pas mal :

```powershell
Get-AppxPackage Microsoft.OneDriveSync | Remove-AppxPackage
Get-AppxPackage MicrosoftWindows.Client.WebExperience | Remove-AppxPackage
```

> [!TIP]
> Remplacez par exemple `Microsoft.OneDriveSync` par ce qui cause l'erreur et relancez sysprep jusqu'à réussite.
> 
> [!WARNING]
> 
> Après 3 essais, il faut réinstaller Windows au complet (limite sysprep)

### 3. Créer la tâche de capture et lancer la capture

1. Normalement l'ordinateur est éteint
2. Dans FOG, créez la tâche de capture
3. Redémarrer l'ordinateur en mode **boot PXE**
4. Ne plus toucher à l'ordinateur, FOG fait le reste
5. À la fin de la capture, l'ordinateur va redémarrer

---

## Auteur

**Alexandre Pareige**

---

## Ressources

- [Site officiel FOG Project](https://fogproject.org/)
- [Documentation FOG Project](https://docs.fogproject.org/)
- [GitHub FOG Project](https://github.com/FOGProject/fogproject)

---

*Dernière mise à jour : 2026*
