https://hedgedoc.cri.epita.fr/s/st3O7RLHk
https://tinyurl.com/exam-reseaux
# VLAN

## Explication
Sur linux on représente les VLANs par des interfaces réseau virtuelles : par exemple, si je souhaite taguer mon trafic sur eth0 sur le VLAN 912, mon interface aura pour nom eth0.912.

Pour creér cette interface virtuelle, il faut faire la commande suivante :
``` 
ip link add link eth0 name eth0.912 type vlan id 912
```

où **912** est le numéro du VLAN et **eth0** le nom de l’interface réseau parente.

Maintenant si je veux mettre l’adresse IP **88.67.108.188/24** sur l’interface virtuelle **eth0.912** :
```
ip address add 88.67.108.188/24 dev eth0.912
```

Maintenant il nous reste à activer cette interface :

``` 
ip link set dev eth0.912 up
```

Éventuellement, si mon routeur par défaut se trouve sur le VLAN **912** et a pour adresse **88.67.108.1** : 

```
ip route add default via 88.67.108.1
```

## Mise en place dans un cas concret
Ces commandes ne sont pas persistées au redémarrage de la machine. La technique pour que ces commandes soient exécutées au démarrage de l’interface est la suivante :

On met toutes les commandes dans un fichier **/etc/network/eth0-up-script** :
```bash
#! /bin/sh
ip link add link eth0 name eth0.912 type vlan id 912
ip address add 88.67.108.188/24 dev eth0.912
ip link set dev eth0.912 up
ip route add default via 88.67.108.1
```

**On peut ajouter autant d'interface VLAN que nécessaire en répétant ces commandes**
Ne pas oublier de rendre le fichier exécutable avec
```
chmod +x eth0-up-script
```
On configure l’interface **eth0** dans **/etc/network/interfaces** :
```
auto eth0
iface eth0 inet static
        address 169.254.169.254 # Adresse bidon, vous pouvez garder la même
        netmask 255.255.255.255 # Netmask bidon, vous pouvez garder le meme
        post-up /etc/network/eth0-up-script
```
Dans l’exemple ci-dessus, c’est la directive **post-up** qui est importante : le script mentionné dans ```post-up``` va être exécuté dès que l’interface est branchée. Ce qui signifie donc que les VLANs seront configurés directement au branchement de l’interface réseau.

# En gros

Créer un fichier qui sera lancé au démarrage : ```/etc/network/eth0-up-script```

Rajouter dedans les ips à ajouter sur le vlan choisi (ici eth0.912) 
Répéter ces commandes pour chaque adresse IP
```bash
#! /bin/sh
ip link add link eth0 name eth0.912 type vlan id 912
ip address add 88.67.108.188/24 dev eth0.912
ip link set dev eth0.912 up
ip route add default via 88.67.108.1
```

Rajouter le fichier ```/etc/network/eth0-up-script``` dans ```/etc/network/interfaces``` : 
```
auto eth0
iface eth0 inet static
        address 169.254.169.254 # Adresse bidon, vous pouvez garder la même
        netmask 255.255.255.255 # Netmask bidon, vous pouvez garder le meme
        post-up /etc/network/eth0-up-script
```
Donner les permissions d'execution avec ```chmod +x```
Puis redémarrer la machine.

# Vérification

Pour être sûr que notre script fonctionne, il suffit de redémarrer la machine et exécuter
```bash
ip -c a
```
pour vérifier si les interfaces se sont bien créées 

## Débug

Si jamais vous ne voyez pas les interfaces après avoir exécuté cette commande, vérifiez tout d'abord que votre script s'execute sans erreurs : 
```bash
./etc/network/eth0-up-script
```
Si après avoir exécuté le script à la main les interfaces apparaissent dans  ```ìp -c a ``` votre problème concerne surement les permissions ; n'oubliez surtout pas de donner les droits d'execution à votre fichier avec ```chmod +x```

# DHCP

Pour configurer un serveur DHCP, il suffit de configurer la machine dans ```/etc/dhcp/dhcpd.conf``` en ayant configuré aussi le ```/etc/network/interfaces``` juste avant.

Voici un exemple d'une configuration DHCP

```bash 
default-lease-time 600; # Durée par défaut du bail DHCP (en secondes)
max-lease-time 7200; # Durée maximum du bail DHCP (en secondes)
authoritative; # spécifie que le serveur fait autorité sur le réseau
# Déclaration d'un bloc subnet pour le sous-réseau 172.16.0.0/16
subnet 172.16.0.0 netmask 255.255.0.0 {
# Ici se situent toutes les options spécifiques à ce subnet
# Les IPs entre 172.16.3.0 (inclus) et 172.16.3.255 (inclus) seront distribuées
# aux clients demandant une IP via DHCP
    range 172.16.3.0 172.16.3.255;
    # Permet d'indiquer au client quel adresse utiliser pour
    # la gateway / routeur par défaut
    option routers 172.16.0.1;
}
## On peut déclarer autant de bloc subnet que nécessaire
````


# Sources

*intro_vlan de **Léo Portemont***
