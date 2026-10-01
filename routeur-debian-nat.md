# Configurer un routeur Linux sous Debian (NAT)

Ce guide transforme une machine Debian en passerelle : les autres machines du réseau local passent par elle pour accéder à Internet. Il couvre l'activation du routage, le NAT avec `iptables`, la sauvegarde des règles et la configuration des machines clientes.

## Sommaire

1. [Principe et schéma](#1-principe-et-schéma)
2. [Configurer les interfaces du routeur](#2-configurer-les-interfaces-du-routeur)
3. [Activer le routage IP](#3-activer-le-routage-ip)
4. [Activer le NAT](#4-activer-le-nat)
5. [Rendre les règles permanentes](#5-rendre-les-règles-permanentes)
6. [Configurer une machine cliente](#6-configurer-une-machine-cliente)
7. [Tester](#7-tester)
8. [Dépannage](#8-dépannage)
9. [Récapitulatif](#9-récapitulatif)

---

## 1. Principe et schéma

Le routeur possède **deux interfaces réseau** :

- une interface **WAN**, côté Internet (dans les exemples : `eth0`) ;
- une interface **LAN**, côté réseau local (dans les exemples : `eth1`).

```
 Internet ── eth0 [ ROUTEUR Debian ] eth1 ── 192.168.168.0/24 ── Machines clientes
                                    192.168.168.2
```

Pour que ça fonctionne, il faut deux choses :

1. **Le routage** : le noyau accepte de transférer les paquets d'une interface à l'autre.
2. **Le NAT (masquerade)** : le routeur remplace l'adresse privée des clients par la sienne quand le trafic sort vers Internet.

| Élément | Valeur dans les exemples |
|---|---|
| Interface vers Internet (WAN) | `eth0` |
| Interface vers le réseau local (LAN) | `eth1` |
| Adresse du routeur sur le LAN | `192.168.168.2` |
| Réseau local | `192.168.168.0/24` |
| Interface de la machine cliente | `ens33` |

Pour connaître le nom de vos interfaces :

```bash
ip -br address
```

Remplacez `eth0`, `eth1` et `ens33` par vos propres noms dans toutes les commandes.

---

## 2. Configurer les interfaces du routeur

Le fichier à connaître est `/etc/network/interfaces`.

```bash
sudo nano /etc/network/interfaces
```

```
# Interface vers Internet (WAN) : adresse reçue automatiquement
auto eth0
iface eth0 inet dhcp

# Interface vers le réseau local (LAN) : adresse fixe
auto eth1
iface eth1 inet static
    address 192.168.168.2/24
```

> Ne mettez **pas** de `gateway` sur l'interface LAN : la passerelle du routeur vers Internet est déjà fournie par l'interface WAN.

Appliquez la configuration :

```bash
sudo systemctl restart networking
ip -br address
```

---

## 3. Activer le routage IP

Par défaut, Linux ne transfère pas les paquets d'une interface à l'autre. Il faut l'autoriser.

Sous **Debian 13**, créez un fichier dédié dans `/etc/sysctl.d/` :

```bash
sudo nano /etc/sysctl.d/90-ipforward.conf
```

Mettez-y :

```
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

> Sur les versions plus anciennes de Debian, ces lignes se mettent dans `/etc/sysctl.conf`.

Chargez la configuration, puis vérifiez :

```bash
sudo sysctl -p /etc/sysctl.d/90-ipforward.conf
cat /proc/sys/net/ipv4/ip_forward
```

La dernière commande doit afficher `1`.

---

## 4. Activer le NAT

Installez `iptables` s'il n'est pas présent :

```bash
sudo apt update
sudo apt install iptables
```

Pour qu'une machine du réseau local accède à Internet à travers la passerelle, ajoutez la règle suivante :

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### À quel réseau correspond l'interface de cette commande ?

L'option `-o eth0` désigne l'interface de **sortie** : celle qui est reliée à **Internet** (le côté WAN du routeur), pas celle du réseau local. Le NAT s'applique aux paquets qui *sortent* par cette interface.

Si vous vous trompez d'interface, les clients n'auront pas Internet. Vérifiez avec :

```bash
ip route
```

La ligne `default via ...` indique l'interface qui mène à Internet.

### Vérifier la règle

```bash
sudo iptables -L -t nat -v
```

Dans la chaîne `POSTROUTING`, une ligne `MASQUERADE` doit apparaître pour `eth0`. Les compteurs de paquets augmentent quand des clients utilisent la passerelle.

---

## 5. Rendre les règles permanentes

Les règles `iptables` disparaissent au redémarrage. Installez `iptables-persistent` pour les conserver :

```bash
sudo apt install iptables-persistent
```

L'installation propose d'enregistrer les règles actuelles : répondez **Oui**. Si vous ajoutez ou modifiez des règles plus tard, enregistrez-les à nouveau :

```bash
sudo netfilter-persistent save
```

Les règles sauvegardées sont stockées dans `/etc/iptables/rules.v4`.

---

## 6. Configurer une machine cliente

Une machine du réseau local doit utiliser le routeur comme **passerelle par défaut**.

### Voir la route actuelle

```bash
ip route
```

Si aucune ligne `default via ...` n'apparaît, la machine ne sait pas par où sortir : c'est la cause la plus fréquente d'un accès Internet qui ne marche pas.

### Ajouter la route par défaut (temporaire)

```bash
sudo ip route add default via 192.168.168.2 dev ens33
```

Remplacez `192.168.168.2` par l'adresse du routeur sur le LAN et `ens33` par l'interface de la machine cliente. Cette route disparaît au redémarrage.

### La rendre permanente

Éditez `/etc/network/interfaces` sur la machine cliente :

```
auto ens33
iface ens33 inet static
    address 192.168.168.10/24
    gateway 192.168.168.2
```

Puis :

```bash
sudo systemctl restart networking
```

Si les adresses sont distribuées par un serveur DHCP, indiquez-lui `192.168.168.2` comme passerelle (`option routers`) : les clients la recevront automatiquement.

---

## 7. Tester

Depuis la machine cliente, testez dans cet ordre :

```bash
ping -c 2 192.168.168.2     # le routeur répond
ping -c 2 8.8.8.8           # Internet par adresse IP
ping -c 2 google.com        # Internet par nom (teste le DNS)
```

| Résultat | Signification |
|---|---|
| Le premier ping échoue | Problème de câblage, d'adresse IP ou de réseau entre le client et le routeur |
| Le deuxième ping échoue | Routage ou NAT mal configuré sur le routeur |
| Le troisième échoue alors que le deuxième marche | Problème de DNS, pas de routage |

Pour le DNS, vérifiez le contenu de `/etc/resolv.conf` sur le client, ou donnez des serveurs DNS via le DHCP.

---

## 8. Dépannage

### Le client n'accède pas à Internet

1. Sur le client, `ip route` : la ligne `default via <ip du routeur>` est-elle présente ? Sinon, ajoutez-la (étape 6).
2. Sur le routeur, `cat /proc/sys/net/ipv4/ip_forward` : doit afficher `1`.
3. Sur le routeur, `sudo iptables -L -t nat -v` : la règle `MASQUERADE` est-elle là, avec la bonne interface de sortie ?
4. Le routeur lui-même a-t-il Internet ? Testez `ping -c 2 8.8.8.8` depuis le routeur.

### Les règles ont disparu après un redémarrage

Elles n'ont pas été sauvegardées. Recréez-les, puis lancez `sudo netfilter-persistent save`.

### Suivre le trafic

```bash
sudo tcpdump -i eth0 -n icmp
```

Pendant que le client fait un `ping 8.8.8.8`, les paquets doivent apparaître avec l'adresse du routeur (traduite par le NAT), pas celle du client.

---

## 9. Récapitulatif

```bash
# 1. Interfaces : /etc/network/interfaces (WAN en dhcp, LAN en statique)

# 2. Routage
sudo nano /etc/sysctl.d/90-ipforward.conf
#   net.ipv4.ip_forward=1
#   net.ipv6.conf.all.forwarding=1
sudo sysctl -p /etc/sysctl.d/90-ipforward.conf
cat /proc/sys/net/ipv4/ip_forward

# 3. NAT
sudo apt install iptables
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -L -t nat -v

# 4. Persistance
sudo apt install iptables-persistent
sudo netfilter-persistent save

# 5. Sur le client
ip route
sudo ip route add default via 192.168.168.2 dev ens33
```
