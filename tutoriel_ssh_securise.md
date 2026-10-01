# Guide : Connexion SSH Rapide et Sécurisée

**À quoi sert ce fichier ?**
Ce document est un tutoriel complet pour sécuriser l'accès à distance d'un serveur Linux. Il permet de remplacer l'authentification classique par mot de passe (vulnérable aux attaques) par une paire de clés cryptographiques modernes (Ed25519). Il explique également comment configurer l'agent SSH local pour ne plus avoir à taper la phrase secrète (passphrase) à chaque connexion, rendant l'accès à la fois extrêmement sécurisé et instantané.

---

## Étape 1 : Préparation du Serveur (Machine Cible)

Connectez-vous à votre serveur cible pour mettre à jour le système et installer le service SSH.

### Commande principale :
```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install openssh-server
```
*Alternative (plus moderne) :* 
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server -y
```

### Vérifier le statut du service :
```bash
sudo systemctl status ssh
```
*Alternative :* `sudo service ssh status` ou `systemctl is-active ssh`

---

## Étape 2 : Génération de la Paire de Clés (Machine Cliente)

Sur votre machine personnelle, créez la paire de clés.

### Commande principale :
```bash
ssh-keygen -t ed25519
```
*Alternative (si la machine cible est très ancienne et ne supporte pas ed25519) :* 
```bash
ssh-keygen -t rsa -b 4096
```

*(Astuce : Validez l'emplacement par défaut avec Entrée. Si vous mettez une passphrase, l'étape 3 l'automatisera).*

---

## Étape 3 : Configuration de l'Agent SSH (Machine Cliente)

Pour mémoriser la clé et éviter les saisies répétitives, nous allons modifier le fichier de comportement de votre terminal.

### Éditer le fichier bashrc :
```bash
nano ~/.bashrc
```
*Alternative :* `vim ~/.bashrc` ou `gedit ~/.bashrc` (si interface graphique).

Ajoutez ce script tout à la fin du fichier :
```bash
SSH_ENV="$HOME/.ssh/environment"

function start_agent {
    echo "Initialising new SSH agent..."
    /usr/bin/ssh-agent | sed 's/^echo/#echo/' > "${SSH_ENV}"
    echo succeeded
    chmod 600 "${SSH_ENV}"
    . "${SSH_ENV}" > /dev/null
    /usr/bin/ssh-add
}

if [ -f "${SSH_ENV}" ]; then
    . "${SSH_ENV}" > /dev/null
    ps -ef | grep ${SSH_AGENT_PID} | grep ssh-agent$ > /dev/null || {
        start_agent;
    }
else
    start_agent;
fi
```

### Appliquer les changements et ajouter la clé :
```bash
source ~/.bashrc
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
*Alternative (pour l'application de `.bashrc`) :* Fermez simplement le terminal et rouvrez-le.

---

## Étape 4 : Transfert de la Clé Publique vers le Serveur

Il faut maintenant copier votre clé publique sur le serveur pour qu'il vous reconnaisse.

### Commande principale (Méthode manuelle expliquée dans votre texte) :
1. Afficher la clé locale :
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
2. Se connecter au serveur, créer le dossier et coller la clé :
   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   nano ~/.ssh/authorized_keys
   ```
   *(Collez la clé copiée, puis sauvegardez)*
3. Sécuriser le fichier :
   ```bash
   chmod 600 ~/.ssh/authorized_keys
   ```

### 🔥 SUPER ALTERNATIVE (Hautement recommandée - remplace toute l'étape 4) :
Plutôt que de tout faire manuellement, utilisez cet utilitaire natif qui copie la clé, crée les dossiers et applique les bons droits (700/600) automatiquement :
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub utilisateur@adresse_ip_du_serveur
```

---

## Étape 5 : Test Final & Dépannage

Testez la connexion depuis votre machine cliente :

```bash
ssh utilisateur@adresse_ip_du_serveur
```
*Alternative (si le port SSH a été changé, ex: port 2222) :*
```bash
ssh -p 2222 utilisateur@adresse_ip_du_serveur
```

**Dépannage :**
* Assurez-vous des droits distants : `700` sur le dossier `~/.ssh` et `600` sur le fichier `authorized_keys`.
* Vérifiez que l'agent tourne : `ssh-add -l` (doit lister votre clé).