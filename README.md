# Mes tutos

Petit site statique qui regroupe mes guides pas à pas. Pas d'outil de compilation : ce sont de simples fichiers HTML, CSS et JavaScript, publiés avec GitHub Pages.

## Structure

```
.
├── index.html              Sommaire avec recherche et filtres
├── assets/
│   ├── style.css           Style partagé par toutes les pages
│   ├── site.js             Copie du code, avancement, onglets, filtre d'erreurs
│   └── tutos.js            Liste des tutos affichés sur le sommaire
├── guides/
│   └── mysql-fedora/       Un dossier par tuto
│       └── index.html
└── _modele/
    └── index.html          Modèle à copier pour un nouveau tuto
```

## Ajouter un tuto

1. Copiez le dossier `_modele/` vers `guides/<mon-slug>/`. Le slug n'a ni espace ni accent, par exemple `proxmox-labo`.
2. Ouvrez `guides/<mon-slug>/index.html` : les commentaires numérotés indiquent quoi changer. Pensez à mettre le même slug dans `data-slug` de la balise `<body>`.
3. Ajoutez un bloc dans `assets/tutos.js` (un exemple est déjà dedans) avec le titre, la description, les thèmes, le niveau, la durée, le nombre d'étapes et la date.
4. Envoyez les fichiers sur GitHub : le tuto apparaît dans le sommaire.

Le sommaire affiche aussi l'avancement du lecteur (par exemple « 3 / 7 ») pour chaque tuto, grâce aux cases « Étape terminée ». Il est enregistré dans son navigateur, rien n'est envoyé à un serveur.

## Voir le site sur son ordinateur

Depuis le dossier du projet :

```bash
python3 -m http.server 8000
```

Puis ouvrez http://localhost:8000.

## Publier avec GitHub Pages

1. Créez un dépôt, par exemple `tutos`, en **Public**.
2. Envoyez-y tout le contenu de ce dossier.
3. Dans **Settings > Pages**, choisissez la branche `main` et le dossier `/ (root)`, puis **Save**.
4. Après une ou deux minutes, le site est à `https://<votre-pseudo>.github.io/tutos/`.

Le dossier `_modele/` commence par un tiret bas : GitHub Pages l'ignore, il n'est donc pas visible sur le site publié. N'ajoutez pas de fichier `.nojekyll`, sinon il serait publié.

## Lier le site à son portfolio

Ajoutez simplement un lien dans la page d'accueil du portfolio :

```html
<a href="https://<votre-pseudo>.github.io/tutos/">Mes tutos</a>
```

Pour le chemin inverse, une ligne est prévue en commentaire dans la barre du haut de `index.html`.
# mysql-fedora-guide
