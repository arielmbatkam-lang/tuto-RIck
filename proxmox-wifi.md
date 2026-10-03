# Connecter un serveur Proxmox VE en WiFi

> Proxmox VE est basé sur **Debian**. La configuration WiFi se fait donc avec les outils Debian classiques (`NetworkManager`, `nmcli`, `wpasupplicant`).

## Prérequis

- Un accès console ou SSH au serveur (au départ, via câble Ethernet).
- Un accès Internet temporaire (câble) pour installer les paquets.
- Une carte WiFi reconnue par le noyau Linux.
- Un réseau WiFi en **WPA2-Personal** (WPA2-PSK).

> ⚠️ **Limite importante** : une carte WiFi en mode client ne peut pas être ajoutée à un bridge Linux (`vmbr0`). Les VM ne pourront donc pas être "bridgées" directement sur le WiFi. Il faut utiliser du **NAT** ou du **routage** pour leur donner accès au réseau.

---

## 1. Identifier la carte réseau WiFi

```bash
lspci -k | grep -A 3 -i network
```

| Élément | Rôle |
|---|---|
| `lspci` | Liste tous les périphériques PCI (cartes réseau, GPU, contrôleurs USB, etc.). |
| `-k` | Affiche, pour chaque périphérique, le **pilote utilisé** (`Kernel driver in use`) et les **modules du noyau disponibles** (`Kernel modules`). |
| `grep -A 3 -i network` | Filtre les lignes contenant "network" (insensible à la casse) et affiche les 3 lignes qui suivent. |

Si la ligne `Kernel driver in use` est absente pour la carte WiFi, le pilote ou le firmware manque (voir l'étape 2).

Pour connaître le **nom de l'interface** (ex. `wlp2s0`, `wlan0`) :

```bash
ip link
```

---

## 2. Installer les paquets nécessaires

```bash
sudo apt update
sudo apt install network-manager wpasupplicant wireless-tools iw rfkill
```

| Paquet | Rôle |
|---|---|
| `network-manager` | Service qui gère les connexions réseau et fournit la commande `nmcli`. |
| `wpasupplicant` | Gère l'authentification WPA/WPA2 auprès de la box. **Indispensable** pour le WiFi sécurisé. |
| `wireless-tools` / `iw` | Outils de diagnostic WiFi (`iwconfig`, `iw dev`, etc.). |
| `rfkill` | Permet de voir/débloquer le blocage logiciel ou matériel de la radio. |

Si la carte n'est pas détectée par le noyau, installer le firmware correspondant (nécessite le dépôt `non-free-firmware` de Debian), par exemple :

```bash
sudo apt install firmware-iwlwifi      # cartes Intel
sudo apt install firmware-realtek      # cartes Realtek
sudo apt install firmware-atheros      # cartes Atheros
```

Puis redémarrer le serveur.

---

## 3. Activer la radio WiFi

```bash
sudo nmcli radio wifi on
```

- `nmcli` : client en ligne de commande de NetworkManager.
- `radio wifi on` : active l'émetteur/récepteur WiFi.

En cas de blocage :

```bash
rfkill list            # affiche l'état des blocages
sudo rfkill unblock wifi
```

---

## 4. Vérifier l'état des périphériques

```bash
nmcli device status
```

Cette commande liste les interfaces et leur état :

| État | Signification |
|---|---|
| `disconnected` | L'interface est gérée par NetworkManager, mais pas connectée. ✅ C'est ce qu'on veut. |
| `unmanaged` | L'interface est ignorée par NetworkManager (souvent car elle est déclarée dans `/etc/network/interfaces`). ❌ À corriger. |
| `connected` | L'interface est connectée. |

### Si la carte WiFi est en `unmanaged`

**a) Vérifier `/etc/network/interfaces`**

```bash
sudo nano /etc/network/interfaces
```

Si l'interface WiFi y est déclarée, **mettre ses lignes en commentaire** avec `#`.

> ⚠️ Ne commentez **pas** les lignes de `lo`, de l'interface Ethernet ni du bridge `vmbr0` : ce sont elles qui assurent l'accès à l'interface web de Proxmox. Faites une sauvegarde avant : `sudo cp /etc/network/interfaces /etc/network/interfaces.bak`.

**b) Vérifier `NetworkManager.conf`**

```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

> Attention à la casse : le chemin est `/etc/NetworkManager/NetworkManager.conf` (majuscules).

Dans la section `[ifupdown]`, mettre :

```ini
[ifupdown]
managed=true
```

> ⚠️ Avec `managed=true`, NetworkManager peut aussi prendre en charge d'autres interfaces déclarées dans `/etc/network/interfaces` et perturber la configuration réseau de Proxmox. Si l'interface WiFi n'est plus listée dans `/etc/network/interfaces`, essayez d'abord **sans** modifier cette option, et ne l'activez que si l'interface reste `unmanaged`.

**c) Redémarrer le service**

```bash
sudo systemctl restart NetworkManager
```

**d) Revérifier**

```bash
nmcli device status
```

L'interface WiFi doit maintenant être en `disconnected`.

---

## 5. Lister les réseaux WiFi détectés

```bash
nmcli device wifi list
```

Affiche les SSID captés, avec leur canal, leur débit, la force du signal et la sécurité (`WPA2`, etc.).

Pour forcer un nouveau scan :

```bash
nmcli device wifi rescan
```

---

## 6. Se connecter au WiFi

```bash
sudo nmcli device wifi connect "NOM_DU_WIFI" password "MOT_DE_PASSE" ifname NOM_INTERFACE_WIFI
```

| Élément | Rôle |
|---|---|
| `device wifi connect` | Crée et active une connexion WiFi. |
| `"NOM_DU_WIFI"` | SSID du réseau (entre **guillemets droits** `"`, pas de guillemets typographiques `“ ”`). |
| `password "..."` | Clé WiFi. |
| `ifname ...` | (Optionnel) Interface à utiliser, utile s'il y a plusieurs cartes WiFi. |

> 🔐 Le réseau WiFi doit être en **WPA2-Personal** (réglable dans l'interface de votre box). Les modes WPA3 seul ou "Enterprise" peuvent nécessiter une configuration supplémentaire.

La connexion est enregistrée et se rétablit automatiquement au redémarrage (`autoconnect`).

---

## 7. Vérifier la connexion

**Adresse IP obtenue :**

```bash
ip addr show NOM_INTERFACE_WIFI
```

Cherchez une ligne `inet 192.168.x.x/24` : c'est l'adresse reçue en DHCP.

**Test de connectivité via l'interface WiFi :**

```bash
ping -I NOM_INTERFACE_WIFI 8.8.8.8
```

- `ping` : envoie des paquets ICMP pour tester l'accessibilité d'une machine.
- `-I` : force l'**interface source** (ou une adresse IP source). Sans cette option, le ping peut passer par le câble Ethernet et fausser le test.
- `8.8.8.8` : serveur DNS public de Google, utilisé comme cible de test.

Variante avec l'adresse IP source :

```bash
ping -I ADRESSE_IP_DU_WIFI 8.8.8.8
```

> Les adresses IP s'écrivent avec des **points** (`8.8.8.8`), pas des virgules.

---

## 8. Gérer la route par défaut

Si le serveur a aussi un câble Ethernet, il peut avoir **deux routes par défaut**. Pour afficher la table de routage :

```bash
ip route
```

**Supprimer la route par défaut :**

```bash
sudo ip route del default
```

- `ip route` : manipule la table de routage.
- `del default` : supprime la route par défaut (`0.0.0.0/0`), c'est-à-dire la passerelle vers Internet.

> ⚠️ Si vous êtes connecté en SSH via le câble, supprimer cette route **coupe votre session**. Faites cette manipulation depuis la console locale, ou via le WiFi.

**Ajouter la route par défaut via le WiFi :**

```bash
sudo ip route add default via IP_DE_LA_BOX dev NOM_INTERFACE_WIFI
```

| Élément | Rôle |
|---|---|
| `add default` | Ajoute une route par défaut. |
| `via IP_DE_LA_BOX` | **Passerelle** (l'adresse IP du routeur/box, ex. `192.168.1.1`), et **non** l'adresse IP du serveur. |
| `dev NOM_INTERFACE_WIFI` | Interface par laquelle le trafic doit sortir. |

La passerelle s'obtient avec `ip route` (ligne `default via ...`) ou :

```bash
nmcli device show NOM_INTERFACE_WIFI | grep IP4.GATEWAY
```

> ⚠️ Ces commandes `ip route` ne sont **pas persistantes** : elles disparaissent au redémarrage.

### Rendre la priorité du WiFi persistante (optionnel)

Avec NetworkManager, on peut régler la priorité des routes via la **métrique** (plus elle est basse, plus la route est prioritaire) :

```bash
sudo nmcli connection modify "NOM_DU_WIFI" ipv4.route-metric 50
sudo nmcli connection up "NOM_DU_WIFI"
```

---

## Récapitulatif des commandes

| Commande | Rôle |
|---|---|
| `lspci -k \| grep -A 3 -i network` | Identifier la carte réseau et son pilote. |
| `ip link` | Lister les interfaces et leurs noms. |
| `sudo apt install network-manager wpasupplicant` | Installer la gestion du WiFi. |
| `sudo nmcli radio wifi on` | Activer la radio WiFi. |
| `nmcli device status` | Voir l'état des interfaces (`unmanaged`, `disconnected`…). |
| `sudo systemctl restart NetworkManager` | Redémarrer NetworkManager après modification de la config. |
| `nmcli device wifi list` | Lister les réseaux WiFi visibles. |
| `sudo nmcli device wifi connect "SSID" password "MDP"` | Se connecter à un réseau WiFi. |
| `ip addr show INTERFACE` | Vérifier l'adresse IP obtenue. |
| `ping -I INTERFACE 8.8.8.8` | Tester la connectivité via l'interface choisie. |
| `ip route` | Afficher la table de routage. |
| `sudo ip route del default` | Supprimer la route par défaut. |
| `sudo ip route add default via IP_BOX dev INTERFACE` | Définir la route par défaut via le WiFi. |

---

## Dépannage rapide

- **Interface WiFi absente de `ip link`** : pilote ou firmware manquant (étape 2), vérifier avec `dmesg | grep -i firmware`.
- **`unmanaged` persistant** : revoir `/etc/network/interfaces` et `NetworkManager.conf` (étape 4).
- **Connecté mais pas d'Internet** : vérifier la route par défaut (`ip route`) et le DNS (`cat /etc/resolv.conf`).
- **Interface web Proxmox inaccessible** : vérifier que `vmbr0` n'a pas été modifié ; l'IP de gestion doit rester joignable.
