# Configurer un serveur DHCP sur Debian

Ce guide installe et configure un serveur DHCP avec `isc-dhcp-server` sur Debian : plage d'adresses, passerelle, DNS, réservation d'adresse et vérifications.

## Sommaire

1. [Schéma et prérequis](#1-schéma-et-prérequis)
2. [Donner une IP fixe au serveur](#2-donner-une-ip-fixe-au-serveur)
3. [Installer le serveur DHCP](#3-installer-le-serveur-dhcp)
4. [Choisir l'interface d'écoute](#4-choisir-linterface-découte)
5. [Configurer le service](#5-configurer-le-service)
6. [Tester la configuration et démarrer](#6-tester-la-configuration-et-démarrer)
7. [Vérifier côté client](#7-vérifier-côté-client)
8. [Pare-feu](#8-pare-feu)
9. [Aller plus loin](#9-aller-plus-loin)
10. [Dépannage](#10-dépannage)
11. [Récapitulatif](#11-récapitulatif)

---

## 1. Schéma et prérequis

Les exemples utilisent ce réseau. Adaptez les adresses au vôtre.

| Élément | Valeur |
|---|---|
| Réseau | `192.168.10.0/24` |
| Passerelle (routeur ou firewall) | `192.168.10.1` |
| Serveur DHCP (cette machine) | `192.168.10.2` |
| Plage distribuée aux clients | `192.168.10.100` à `192.168.10.200` |
| Interface réseau du serveur | `ens33` |
| Serveurs DNS donnés aux clients | `1.1.1.1` et `9.9.9.9` |

Il vous faut :

- Une machine Debian avec un accès `sudo`.
- Une interface réseau reliée au réseau à desservir.
- **Un seul serveur DHCP actif sur ce réseau.** Si un routeur ou un firewall (box, pfSense, etc.) distribue déjà des adresses, désactivez son DHCP, sinon les deux serveurs se concurrencent.

Pour connaître le nom de votre interface :

```bash
ip -br address
```

Le nom (`ens33`, `eth0`, `enp0s3`...) apparaît dans la première colonne. Remplacez `ens33` par le vôtre dans toutes les commandes suivantes.

---

## 2. Donner une IP fixe au serveur

Un serveur DHCP doit avoir une adresse fixe, il ne peut pas se la demander à lui-même.

### Avec `/etc/network/interfaces` (installation Debian classique)

```bash
sudo nano /etc/network/interfaces
```

```
auto ens33
iface ens33 inet static
    address 192.168.10.2/24
    gateway 192.168.10.1
```

Appliquez la configuration :

```bash
sudo systemctl restart networking
```

> Si vous êtes connecté en SSH, la connexion peut se couper au redémarrage du réseau. Reconnectez-vous à la nouvelle adresse.

### Avec NetworkManager (environnement de bureau)

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.10.2/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns "1.1.1.1 9.9.9.9"
sudo nmcli connection up "Wired connection 1"
```

Le nom de la connexion s'affiche avec `nmcli connection show`.

### Vérifier

```bash
ip -br address show ens33
ping -c 2 192.168.10.1
```

---

## 3. Installer le serveur DHCP

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

Ne vous inquiétez pas si l'installation affiche une erreur de démarrage du service : il n'est pas encore configuré, c'est normal.

> **Remarque :** ISC DHCP n'est plus développé activement. Il reste disponible dans Debian et convient très bien pour un réseau local ou un laboratoire. Son successeur de l'éditeur est **Kea** (`kea-dhcp4-server`), dont la configuration est différente (fichier JSON).

---

## 4. Choisir l'interface d'écoute

Indiquez au service sur quelle interface écouter les demandes :

```bash
sudo nano /etc/default/isc-dhcp-server
```

```
INTERFACESv4="ens33"
INTERFACESv6=""
```

Laissez `INTERFACESv6` vide si vous ne distribuez que de l'IPv4.

---

## 5. Configurer le service

Sauvegardez le fichier d'origine, qui contient de nombreux exemples commentés, puis écrivez le vôtre :

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.orig
sudo nano /etc/dhcp/dhcpd.conf
```

```
# Options communes à tous les réseaux
option domain-name "lab.local";
option domain-name-servers 1.1.1.1, 9.9.9.9;

# Durée des baux, en secondes (10 minutes par défaut, 2 heures au maximum)
default-lease-time 600;
max-lease-time 7200;

# Ce serveur fait autorité sur ce réseau
authoritative;

# Le réseau desservi
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option broadcast-address 192.168.10.255;
}
```

### Réserver une adresse pour un appareil

Une réservation donne toujours la même adresse à un appareil, repéré par son adresse MAC. Ajoutez ce bloc à la suite du fichier :

```
host imprimante {
    hardware ethernet AA:BB:CC:DD:EE:FF;
    fixed-address 192.168.10.50;
}
```

Choisissez une adresse **hors de la plage** `range`, ici `.50` (la plage commence à `.100`).

Pour trouver l'adresse MAC d'un client Linux :

```bash
ip link show
```

Elle apparaît après `link/ether`. Sous Windows, utilisez `ipconfig /all` (ligne « Adresse physique »).

### Explication des paramètres

| Paramètre | Rôle |
|---|---|
| `range` | Première et dernière adresse distribuées automatiquement |
| `option routers` | Passerelle donnée aux clients |
| `option domain-name-servers` | Serveurs DNS donnés aux clients |
| `default-lease-time` | Durée normale d'un bail |
| `max-lease-time` | Durée maximale qu'un client peut demander |
| `authoritative` | Ce serveur répond de façon définitive aux demandes de ce réseau |
| `host` ... `fixed-address` | Réservation d'une adresse pour une adresse MAC |

---

## 6. Tester la configuration et démarrer

Vérifiez d'abord que le fichier ne contient pas d'erreur :

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Aucune sortie d'erreur signifie que la syntaxe est correcte. Puis démarrez le service :

```bash
sudo systemctl enable --now isc-dhcp-server
sudo systemctl status isc-dhcp-server --no-pager
```

Le statut doit afficher `active (running)`. Après chaque modification de `dhcpd.conf` :

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf && sudo systemctl restart isc-dhcp-server
```

---

## 7. Vérifier côté client

### Sur un client Linux

```bash
sudo dhclient -r ens33
sudo dhclient -v ens33
ip -br address show ens33
```

Le client doit recevoir une adresse entre `192.168.10.100` et `192.168.10.200`.

### Sur un client Windows

```
ipconfig /release
ipconfig /renew
ipconfig /all
```

La ligne « Serveur DHCP » doit afficher `192.168.10.2`.

### Côté serveur

Voir les baux attribués :

```bash
sudo cat /var/lib/dhcp/dhcpd.leases
```

Suivre les échanges en direct :

```bash
sudo journalctl -u isc-dhcp-server -f
```

Les lignes `DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST` puis `DHCPACK` montrent une attribution réussie.

---

## 8. Pare-feu

Debian n'active pas de pare-feu par défaut. Si vous utilisez `ufw`, autorisez le port DHCP :

```bash
sudo ufw allow 67/udp
```

Avec `nftables`, autorisez le trafic UDP entrant sur le port `67` pour l'interface `ens33`.

---

## 9. Aller plus loin

### Plusieurs réseaux (VLANs)

Ajoutez une interface d'écoute par réseau dans `/etc/default/isc-dhcp-server` :

```
INTERFACESv4="ens33 ens33.20"
```

Puis déclarez un bloc `subnet` par réseau dans `dhcpd.conf`. **Chaque interface doit avoir une adresse dans le réseau du `subnet` correspondant.**

### Desservir des réseaux distants

Un routeur doit relayer les demandes DHCP vers le serveur (« DHCP relay » ou « IP helper »), car les demandes DHCP ne traversent pas les routeurs.

### Donner des options supplémentaires

```
option ntp-servers 192.168.10.1;
option domain-search "lab.local";
```

---

## 10. Dépannage

### Le service refuse de démarrer

```bash
sudo journalctl -u isc-dhcp-server -n 30 --no-pager
```

Les causes les plus fréquentes :

| Message | Cause et correction |
|---|---|
| `No subnet declaration for ens33 (192.168.x.x)` | L'adresse IP de l'interface n'est pas dans un `subnet` déclaré. Vérifiez l'IP fixe de l'étape 2 et le bloc `subnet`. |
| `Not configured to listen on any interfaces!` | `INTERFACESv4` est vide ou le nom d'interface est faux. |
| `semicolon expected` ou `parse error` | Un `;` manque, ou une accolade n'est pas fermée dans `dhcpd.conf`. Lancez `dhcpd -t`. |
| `Can't open /var/lib/dhcp/dhcpd.leases: Permission denied` | Rétablissez les droits : `sudo chown dhcpd:dhcpd /var/lib/dhcp/dhcpd.leases`. |

### Les clients ne reçoivent pas d'adresse

- Le client est-il sur le même réseau (même switch, même VLAN, même bridge de machine virtuelle) que le serveur ?
- Un autre serveur DHCP répond-il ? Un client qui reçoit une adresse inattendue en est le signe.
- Les demandes arrivent-elles au serveur ? Vérifiez avec :

```bash
sudo tcpdump -i ens33 -n port 67 or port 68
```

### Une adresse réservée n'est pas attribuée

Vérifiez l'adresse MAC (casse et `:` sans importance, mais chaque octet doit être exact), puis redémarrez le service. Le client doit renouveler son bail (`dhclient -r` puis `dhclient`).

---

## 11. Récapitulatif

```bash
# IP fixe sur le serveur (voir étape 2), puis :
sudo apt update
sudo apt install isc-dhcp-server

# Interface d'écoute
sudo nano /etc/default/isc-dhcp-server      # INTERFACESv4="ens33"

# Configuration du réseau
sudo nano /etc/dhcp/dhcpd.conf

# Test et démarrage
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl enable --now isc-dhcp-server
sudo systemctl status isc-dhcp-server --no-pager

# Suivi
sudo journalctl -u isc-dhcp-server -f
sudo cat /var/lib/dhcp/dhcpd.leases
```
