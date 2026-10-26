# Notes techniques — blog GitHub Pages

Site Jekyll (thème `minima`) prêt à publier sur GitHub Pages.

## Contenu

```
_config.yml                    Réglages du site (titre, URL, thème)
index.md                       Page d'accueil : liste automatique des articles
about.md                       Page « À propos »
_posts/                        Les articles, nommés AAAA-MM-JJ-titre.md
assets/images/                 Images et schémas (SVG)
Gemfile                        Uniquement pour prévisualiser en local
```

## Publier en 4 étapes

1. **Personnalisez `_config.yml`** : remplacez `<votre-nom>` par votre identifiant GitHub et `<Votre nom>` par votre nom.
   - Dépôt nommé `<votre-nom>.github.io` → laissez `baseurl: ""`.
   - Autre nom (ex. `blog-auth`) → mettez `baseurl: "/blog-auth"`.
2. **Créez le dépôt** sur GitHub (public), puis poussez les fichiers :

   ```bash
   cd blog-auth
   git init
   git add .
   git commit -m "Premier article : protocoles d'authentification"
   git branch -M main
   git remote add origin https://github.com/<votre-nom>/<nom-du-depot>.git
   git push -u origin main
   ```

3. **Activez Pages** : *Settings* → *Pages* → *Source : Deploy from a branch* → branche `main`, dossier `/ (root)` → *Save*.
4. **Patientez 1 à 2 minutes** : l'adresse du site s'affiche en haut de la page *Settings → Pages*.

## Ajouter un article

Créez un fichier dans `_posts/` nommé `AAAA-MM-JJ-mon-titre.md` qui commence par :

```yaml
---
layout: post
title: "Mon titre"
---
```

Poussez-le : le site se reconstruit automatiquement.

## Prévisualiser en local (facultatif)

Avec Ruby installé :

```bash
bundle install
bundle exec jekyll serve
```

Puis ouvrez http://localhost:4000.
