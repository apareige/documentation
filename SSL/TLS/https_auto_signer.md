# Créé un certificat SSL/TLS auto-signé 

- [Créé un certificat SSL/TLS auto-signé](#créé-un-certificat-ssltls-auto-signé)
  - [Activez le module SSL d’apache2](#activez-le-module-ssl-dapache2)
    - [exemple de configuration pour le VirtualHost SSL](#exemple-de-configuration-pour-le-virtualhost-ssl)
  - [créer l’autorité de certification](#créer-lautorité-de-certification)
    - [utilisation, la création n’est faite qu’une fois.](#utilisation-la-création-nest-faite-quune-fois)
    - [création de la clef privée de l’autorité de certification](#création-de-la-clef-privée-de-lautorité-de-certification)
    - [Maintenant qu'on a la clé, on va générer le root certificate (certificat racine).](#maintenant-quon-a-la-clé-on-va-générer-le-root-certificate-certificat-racine)
    - [Dans le cas où vous auriez besoin d’un fichier .crt (au lieu du .pem), exécutez cette commande :](#dans-le-cas-où-vous-auriez-besoin-dun-fichier-crt-au-lieu-du-pem-exécutez-cette-commande-)
    - [créez ensuite une clé privé pour le site wordpress :](#créez-ensuite-une-clé-privé-pour-le-site-wordpress-)
    - [Ensuite, il faut préparer une demande de certificat (CSR(Certificate Signing Request))](#ensuite-il-faut-préparer-une-demande-de-certificat-csrcertificate-signing-request)
    - [Enfin, l’autorité signe la clef du serveur :](#enfin-lautorité-signe-la-clef-du-serveur-)
    - [On va maintenant signer le certificat avec notre autorité de certification, pour une validité de 10.000 jours :](#on-va-maintenant-signer-le-certificat-avec-notre-autorité-de-certification-pour-une-validité-de-10000-jours-)

## Activez le module SSL d’apache2

```bash
sudo a2enmod ssl
```
``` bash
sudo nano /etc/apache2/sites-available/wordpress.conf
```
### exemple de configuration pour le VirtualHost SSL
```bash
ServerName localhost
<VirtualHost *:80>
    UseCanonicalName Off
    ServerAdmin  webmaster@localhost
    DocumentRoot /var/www/wordpress
</VirtualHost>
<IfModule mod_ssl.c>
        <VirtualHost *:443>
                ServerName #URL#     
                SSLEngine on
                ServerAdmin  webmaster@localhost
                DocumentRoot /var/www/wordpress
                <Directory /var/www/wordpress>
                        Options +FollowSymLinks
                        Options -Indexes
                        AllowOverride All
                        order allow,deny
                        allow from all
                </Directory>
        SSLCertificateFile /opt/ca/XXX.crt
        SSLCertificateKeyFile /opt/ca/XXX.key
        SSLCACertificateFile /opt/ca/XXX.crt
        </VirtualHost>
</IfModule>
<Directory /var/www/wordpress>
    Options +FollowSymLinks
    Options -Indexes
    AllowOverride All
    order allow,deny
    allow from all
</Directory>
```
```bash
sudo systemctl restart apache2
```

## créer l’autorité de certification

Les fichiers nécessaires pour chiffrer la communication sont :
- pfXX.key : la cle privé du site
- in pfXX.csr : la demande de certificat ;
- out pfXX.crt : le certificat créé et certifié par l’autorité ;
- CA caXX.crt : le certificat de l’autorité de certification XXX ;
- CAkey caXX.key : la clef privée de l’autorité de certification XXX ;
- CAcreateserial -CAserial caFLD.srl : il faut créer et utiliser un numéro de série qui sera incrémenté à chaque nouvelle

### utilisation, la création n’est faite qu’une fois.

```bash
mkdir /opt/ca
cd /opt/ca
```
### création de la clef privée de l’autorité de certification
```bash
openssl genrsa -out caXX.key 4096
```
### Maintenant qu'on a la clé, on va générer le root certificate (certificat racine). 

Créez ensuite le certificat de l’autorité qui sera importé par le navigateur des clients: caXX.pem

```bash
openssl req -x509 -new -nodes -key caXX.key -sha256 -days 10000 -out caXX.pem
```
```bash
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:Bretagne
Locality Name (eg, city) []:Lannion
Organization Name (eg, company) [Internet Widgits Pty Ltd]:XXX
Organizational Unit Name (eg, section) []:Lycee
Common Name (e.g. server FQDN or YOUR name) []:#URL#
Email Address []:admin@localhost
```
### Dans le cas où vous auriez besoin d’un fichier .crt (au lieu du .pem), exécutez cette commande :
```bash
openssl x509 -in caXXX.pem -inform PEM -out caXXX.crt
```

### créez ensuite une clé privé pour le site wordpress :
```bash
openssl genrsa 4096 > XXXX.key
```

### Ensuite, il faut préparer une demande de certificat (CSR(Certificate Signing Request)) 
- cette demande doit être envoyée à l’autorité de certification :

```bash
openssl req -new -key pfXX.key >pfXX.csr
```

```bash
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:FR
State or Province Name (full name) [Some-State]:Bretagne
Locality Name (eg, city) []:XXX
Organization Name (eg, company) [Internet Widgits Pty Ltd]:XXX
Organizational Unit Name (eg, section) []:XXX
Common Name (e.g. server FQDN or YOUR name) [] : pfXX.pfsio1.bts-sio.local
Email Address []:admin@localhost
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:Password123
An optional company name []:XXX
```

### Enfin, l’autorité signe la clef du serveur :
On va créer un fichier de configuration pour le sous-domaine concerné par l’autorité de certification, pour cela créez le
fichier ca.ext, ainsi complété :
```bash
nano caFLD.ext
```
```bash
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
#DNS.1=FQDN CA, FQDN sites autant que de sites à certifier
DNS.1 = #URL#
#ip des sites à certifier
IP.1 = XXX.XXX.XXX.XXX
```

### On va maintenant signer le certificat avec notre autorité de certification, pour une validité de 10.000 jours :
```bash
openssl x509 -req -in pfXX.csr -CA caFLD.pem -CAkey caFLD.key -CAcreateserial -out pfXX.crt -days 10000 -sha256 -extfile caFLD.ext
```
```bash
systemctl restart apache2
```