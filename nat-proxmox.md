# Réseau NAT sous Proxmox : commandes détaillées et configuration adaptée au labo

Réseau NAT : `10.10.10.0/24` (gateway `10.10.10.1`), bridge **`vmbr2`**, sortie Internet par **`wlp3s0`** (carte Wi-Fi qui porte la route par défaut).

## 1. Principe

```text
VM (10.10.10.x) → vmbr2 (10.10.10.1) → NAT/MASQUERADE → wlp3s0 → box → Internet
```

Trois éléments sont nécessaires :

1. Un **bridge sans port physique** (`bridge-ports none`) qui sert de réseau privé et de passerelle.
2. Le **routage IPv4** activé sur l'hôte.
3. Une règle **MASQUERADE** vers l'interface qui a vraiment Internet.

## 2. Préparer le terrain

Dans Node → Shell :

```bash
# Sauvegarde du fichier actuel
cp /etc/network/interfaces /etc/network/interfaces.bak

# Voir par où sort Internet (cherche la ligne "default")
ip route

# Lister les interfaces réseau (repère le nom wl...)
ip -br link
```

Si une zone SDN a été créée pendant les essais, la supprimer d'abord (Datacenter → SDN → VNets → Remove, puis Zones → Remove, puis **Apply**), pour ne pas avoir deux NAT en parallèle.

## 3. Éditer `/etc/network/interfaces`

```bash
nano /etc/network/interfaces
```

Dans `nano` : `Ctrl+O` puis `Entrée` pour enregistrer, `Ctrl+X` pour quitter.

| Ligne | Rôle |
|---|---|
| `address 10.10.10.1/24` | IP du bridge, qui devient la **gateway des VMs** |
| `bridge-ports none` | bridge purement interne, sans carte physique |
| `bridge-stp off` / `bridge-fd 0` | désactive le spanning tree et le délai de forwarding |
| `post-up echo 1 > /proc/sys/net/ipv4/ip_forward` | active le routage entre `vmbr2` et `wlp3s0` |
| `post-up iptables -t nat -A POSTROUTING ... -j MASQUERADE` | cache les IP privées derrière l'IP Wi-Fi de l'hôte |
| `post-down iptables -t nat -D POSTROUTING ...` | supprime la règle quand le bridge s'arrête |

Détail de la règle iptables :

- `-t nat` : table NAT.
- `-A POSTROUTING` : s'applique aux paquets sur le point de sortir.
- `-s 10.10.10.0/24` : uniquement ceux venant du réseau privé.
- `-o wlp3s0` : qui sortent par le Wi-Fi.
- `-j MASQUERADE` : l'adresse source est remplacée par celle de l'hôte.

## 4. Appliquer

```bash
ifreload -a
```

Aucune erreur ne doit s'afficher. À préférer à `systemctl restart networking`, qui coupe tout le réseau d'un coup : Proxmox est administré **via le Wi-Fi**, une erreur sur `wlp3s0` peut couper l'accès à l'interface web.

## 5. Vérifier côté Proxmox

```bash
ip a show vmbr2                    # doit afficher UP et 10.10.10.1/24
sysctl net.ipv4.ip_forward         # doit afficher = 1
iptables -t nat -S POSTROUTING     # doit montrer MASQUERADE avec -o wlp3s0
```

## 6. Connecter une VM au NAT

Proxmox → VM → **Hardware** → Network Device → **Bridge = `vmbr2`**. Décocher **Firewall** et **Disconnect** le temps des tests.

## 7. Configurer l'IP de la VM (pas de DHCP sur ce bridge)

| Paramètre | Valeur |
|---|---|
| IP | `10.10.10.10/24` (puis `.20`, `.30`...) |
| Masque | `255.255.255.0` |
| Gateway | `10.10.10.1` |
| DNS | `1.1.1.1` |

Exemple sur une VM Debian, dans `/etc/network/interfaces` (adapter `ens18` à l'interface réelle, visible avec `ip -br link`) :

```text
auto ens18
iface ens18 inet static
    address 10.10.10.10/24
    gateway 10.10.10.1
    dns-nameservers 1.1.1.1
```

```bash
systemctl restart networking
```

Pour OPNsense (VM 107), le WAN se règle avec l'option **2** de la console : `10.10.10.2/24`, gateway `10.10.10.1`.

## 8. Tester depuis la VM, dans l'ordre

```bash
ping -c 4 10.10.10.1     # joint l'hôte → le bridge et le câblage sont bons
ping -c 4 1.1.1.1        # sort sur Internet → le NAT marche
ping -c 4 google.com     # le DNS marche
```

Pour confirmer que ça passe par le Wi-Fi, sur l'hôte pendant le ping :

```bash
ip -s link show wlp3s0
tcpdump -ni wlp3s0 icmp
```

## 9. En cas d'échec

- **Pare-feu Proxmox** : décocher Firewall sur la carte réseau de la VM, vérifier qu'il n'est pas actif au niveau Datacenter ou Node (Firewall → Options).
- **MAC filter** : si le firewall est utilisé, le désactiver sur l'interface de la VM.
- **Forwarding** : `iptables -L FORWARD -v -n` doit montrer une politique `ACCEPT`.
- **Retour arrière** :

```bash
cp /etc/network/interfaces.bak /etc/network/interfaces
ifreload -a
```

## 10. Redirection de port (DNAT)

C'est l'inverse du MASQUERADE : une connexion entrante est renvoyée vers une VM. Exemple : le port **33891** de l'hôte vers le RDP (`3389`) d'une VM Windows en `10.10.10.10`.

À ajouter dans le bloc `vmbr2`, à la suite des autres `post-up`/`post-down` :

```text
    post-up   iptables -t nat -A PREROUTING -i wlp3s0 -p tcp --dport 33891 -j DNAT --to-destination 10.10.10.10:3389
    post-down iptables -t nat -D PREROUTING -i wlp3s0 -p tcp --dport 33891 -j DNAT --to-destination 10.10.10.10:3389
```

Puis `ifreload -a`, et test depuis un autre PC du même réseau : `10.17.18.18:33891`.

L'hôte est lui-même derrière un autre routeur (réseau `10.17.18.0/24`) : cette redirection n'est joignable que depuis ce réseau, pas depuis Internet, sauf redirection du port sur le routeur en amont. Un VPN reste préférable pour l'accès à distance.

## 11. Fichier complet à coller dans `/etc/network/interfaces`

Reconstitué d'après les captures : `enp2s0` + `vmbr0` (`192.168.100.2/24`), `vmbr1` VLAN-aware, `wlp3s0` en DHCP. **À comparer avec le fichier actuel** (`cat /etc/network/interfaces.bak`) : garder les lignes existantes si elles diffèrent (contenu exact de `vmbr1` et lignes Wi-Fi non vus).

```text
auto lo
iface lo inet loopback

# Carte filaire (sans IP, rattachée à vmbr0)
iface enp2s0 inet manual

# Wi-Fi : porte la route par défaut et l'accès Internet
auto wlp3s0
iface wlp3s0 inet dhcp
    wpa-ssid NOM_DU_WIFI
    wpa-psk MOT_DE_PASSE

# Bridge filaire (réseau 192.168.100.0/24, sans sortie Internet)
auto vmbr0
iface vmbr0 inet static
    address 192.168.100.2/24
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0

# Bridge interne VLAN-aware (à adapter à la config actuelle)
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Bridge NAT pour les VMs : gateway 10.10.10.1, sortie par le Wi-Fi
auto vmbr2
iface vmbr2 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up   iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlp3s0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o wlp3s0 -j MASQUERADE

source /etc/network/interfaces.d/*
```

Points d'attention :

- Aucune ligne `bridge-...` ne doit apparaître sous `wlp3s0` (cause de l'erreur `ifreload` rencontrée).
- Pas de `gateway` sur `vmbr0` : la route par défaut vient du DHCP du Wi-Fi.
- Faire la sauvegarde de l'étape 2 avant `ifreload -a`, et garder un écran et un clavier branchés sur la machine en cas de coupure.
