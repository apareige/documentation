# Sécurisation contre l’IP Spoofing

## Passer en mode administrateur
```text
Switch> en
```

## Se mettre en mode terminal
```text
Switch# conf t
```

## Empêcher l’usurpation d’adresses IP
```text
Switch(conf t)> interface range gigabitEthernet 1/0/1-28
Switch(if-range)> ip verify source port-security
```

## Affichage

### Afficher les machines bloquées
```text
Switch# show ip verify source
```

### Voir les machines connectées sur le commutateur
```text
Switch# show ip source binding
```
