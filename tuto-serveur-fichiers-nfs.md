# Installation d'un serveur de fichiers NFS sous Linux

Ce tutoriel explique comment installer un serveur de fichiers sous Linux avec **NFS**, et comment y accéder depuis une machine cliente.

## Prérequis

- Une machine Linux (Debian/Ubuntu dans ce tutoriel) avec **au moins 2 disques durs** : un pour le système, un pour les données partagées.
- Le système installé et à jour :

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Partie 1 : Configuration du serveur de fichiers

### 1. Repérer le deuxième disque

Branchez le deuxième disque, puis listez les disques et leurs partitions :

```bash
lsblk
```

Notez le nom du disque (par exemple `sdb`). Vérifiez qu'il ne contient pas déjà de partitions importantes.

### 2. Créer une partition

Lancez `fdisk` sur le disque repéré (exemple avec `/dev/sdb`) :

```bash
sudo fdisk /dev/sdb
```

Saisissez dans l'ordre :

| Touche | Action |
|--------|--------|
| `n` | Nouvelle partition |
| `p` | Partition primaire |
| `Entrée` | Numéro de partition par défaut |
| `Entrée` | Premier secteur par défaut |
| `Entrée` | Dernier secteur par défaut (tout le disque) |
| `w` | Écrire les changements et quitter |

> ⚠️ Attention : cette opération modifie la table de partitions. Vérifiez bien le nom du disque avant de continuer.

### 3. Formater la partition

```bash
sudo mkfs.ext4 /dev/sdb1
```

### 4. Créer le point de montage et monter le disque

Créez d'abord le dossier sur lequel le disque sera monté :

```bash
sudo mkdir /mnt/disquedur
```

Puis montez la partition :

```bash
sudo mount /dev/sdb1 /mnt/disquedur
```

Pour voir tous les systèmes de fichiers montés :

```bash
findmnt
```

> 💡 Ce montage est temporaire : il disparaît au redémarrage. Pour le rendre permanent, ajoutez une ligne dans `/etc/fstab` :
> ```
> /dev/sdb1  /mnt/disquedur  ext4  defaults  0  2
> ```

### 5. Installer les paquets NFS

```bash
sudo apt install nfs-kernel-server nfs-common
```

> Le paquet `portmap` est obsolète, il a été remplacé par `rpcbind`, installé automatiquement comme dépendance.

### 6. Créer le dossier à partager

```bash
sudo mkdir /mnt/disquedur/partage
sudo chmod 777 /mnt/disquedur/partage
```

> `chmod 777` donne tous les droits à tout le monde : acceptable pour un labo, à restreindre en production.

### 7. Déclarer le partage dans `/etc/exports`

Le fichier `/etc/exports` contient la liste des dossiers partagés. On le modifie avec `nano` :

```bash
sudo nano /etc/exports
```

Syntaxe d'une ligne :

```
/chemin/du/dossier   machines_autorisées(options)
```

- **machines_autorisées** : `*` pour tout le monde, ou une adresse IP / un réseau (ex. `192.168.1.0/24`).
- **options** : `rw` (lecture/écriture) ou `ro` (lecture seule), `sync`, etc.

Exemple :

```
/mnt/disquedur/partage  *(rw,sync,no_subtree_check)
```

### 8. Appliquer la configuration

```bash
sudo exportfs -ra
```

Redémarrez le service et vérifiez qu'il fonctionne :

```bash
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

Pour afficher les dossiers actuellement exportés :

```bash
sudo exportfs -v
```

---

## Partie 2 : Configuration du client

### 1. Mettre à jour la machine

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Installer le client NFS

```bash
sudo apt install nfs-common
```

### 3. Créer le point de montage

```bash
sudo mkdir /mnt/partage
```

### 4. Monter le partage

```bash
sudo mount -t nfs -o vers=4,rw IP_DU_SERVEUR:/mnt/disquedur/partage /mnt/partage
```

- Remplacez `IP_DU_SERVEUR` par l'adresse IP du serveur de fichiers.
- Utilisez `rw` pour un accès en lecture/écriture, ou `ro` pour lecture seule.

### 5. Vérifier

```bash
findmnt /mnt/partage
ls /mnt/partage
```

Testez en créant un fichier côté client, puis vérifiez sa présence côté serveur :

```bash
touch /mnt/partage/test.txt
```

> 💡 Pour un montage permanent côté client, ajoutez dans `/etc/fstab` :
> ```
> IP_DU_SERVEUR:/mnt/disquedur/partage  /mnt/partage  nfs  defaults  0  0
> ```

---

## Récapitulatif des commandes

| Étape | Commande |
|-------|----------|
| Lister les disques | `lsblk` |
| Partitionner | `sudo fdisk /dev/sdb` |
| Formater | `sudo mkfs.ext4 /dev/sdb1` |
| Monter le disque | `sudo mount /dev/sdb1 /mnt/disquedur` |
| Voir les montages | `findmnt` |
| Installer NFS (serveur) | `sudo apt install nfs-kernel-server nfs-common` |
| Éditer les partages | `sudo nano /etc/exports` |
| Appliquer les partages | `sudo exportfs -ra` |
| Installer NFS (client) | `sudo apt install nfs-common` |
| Monter le partage (client) | `sudo mount -t nfs -o vers=4,rw IP:/chemin /point_de_montage` |
