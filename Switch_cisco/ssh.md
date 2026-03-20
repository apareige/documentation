#  Sécurisation des accès SSH 

## 1. Configuration de base SSH
```bash
Switch(config)# ip domain-name bts-sio.local
Switch(config)# crypto key generate rsa general-keys modulus 1024
```
## 2. Activation et paramétrage SSH
```bash
Switch(config)# ip ssh version 2
Switch(config)# ip ssh logging events
Switch(config)# ip ssh time-out 60
Switch(config)# ip ssh authentication-retries 3
```
## 3. Sécurisation des lignes VTY
```bash
Switch(config)# ip access-list extended SSH_ACCESS
Switch(config)# permit tcp 172.17.1.2 255.255.255.255 any eq 22
Switch(config)# aaa new-model
Switch(config)# aaa authentication login default local
Switch(config)# line vty 0 2
Switch(config-line)# access-class SSH_ACCESS in
Switch(config-line)# login local
Switch(config-line)# transport input ssh
Switch(config-line)# transport output ssh
Switch(config)# line vty 3 15
Switch(config-line)# transport input none
```