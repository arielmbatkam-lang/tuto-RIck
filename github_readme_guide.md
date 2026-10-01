# Guide de Configuration & Dépannage Réseau : Routeur & Serveur DHCP

Ce dépôt regroupe la documentation, les étapes et les commandes validées pour configurer, nettoyer et dépanner un routeur Linux et un serveur DHCP (`isc-dhcp-server`).

---

## 🎯 Objectif global
Le but de ce guide est de vous accompagner pas à pas dans :
1. La **purge et la réinitialisation** des règles de filtrage et de NAT (`iptables`).
2. La **configuration et l'activation** d'un serveur DHCP sur une interface réseau spécifique.
3. La **validation de la syntaxe** et le démarrage des services systemd.

---

## 🛠️ Récapitulatif des commandes validées

### 1. Nettoyage et Réinitialisation des Règles `iptables`
Ces commandes permettent de purger l'ensemble des règles de filtrage, de NAT et de réinitialiser les politiques par défaut afin de repartir sur une configuration propre.

* **Vider toutes les tables et chaînes :**
  ```bash
  sudo iptables -F
  sudo iptables -t nat -F
  sudo iptables -t mangle -F
  sudo iptables -X
  ```
* **Rétablir les politiques par défaut (Autoriser tout le trafic) :**
  ```bash
  sudo iptables -P INPUT ACCEPT
  sudo iptables -P FORWARD ACCEPT
  sudo iptables -P OUTPUT ACCEPT
  ```

---

### 2. Configuration de l'Interface du Serveur DHCP
Indique au démon DHCP sur quelle interface réseau physique ou virtuelle il doit écouter et distribuer les baux IP.

* **Édition du fichier de configuration par défaut :**
  ```bash
  sudo nano /etc/default/isc-dhcp-server
  ```
  *(Assurez-vous d'y renseigner votre interface, par exemple : `INTERFACESv4="ens37"`)*

---

### 3. Validation de la Syntaxe du DHCP
Vérifie l'intégrité du fichier de configuration principal (`/etc/dhcp/dhcpd.conf`) avant de lancer le service pour éviter tout plantage au démarrage.

* **Test de syntaxe de configuration :**
  ```bash
  sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
  ```

---

### 4. Démarrage et Activation du Service DHCP
Active le service pour qu'il se lance automatiquement au démarrage du système et le démarre immédiatement dans la session en cours.

* **Activation et lancement du service systemd :**
  ```bash
  sudo systemctl enable --now isc-dhcp-server
  ```

---

### 5. Vérification de l'État du Service
Permet de contrôler si le serveur DHCP est bien actif, s'il écoute correctement sur l'interface définie et d'analyser d'éventuelles erreurs en temps réel.

* **Affichage du statut complet sans pagination :**
  ```bash
  sudo systemctl status isc-dhcp-server --no-pager