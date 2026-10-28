# Notes techniques — blog GitHub Pages

Site Jekyll (thème `minima`) prêt à publier sur GitHub Pages.

## Contenu

```
_config.yml                    Réglages du site (titre, URL, thème)
index.html                     Page d'accueil : liste des articles en français
en/index.html                  Page d'accueil anglaise (/en/) : liste des articles en anglais
about.md                       Page « À propos »
_posts/                        Les articles (FR et EN), nommés AAAA-MM-JJ-titre.md
_includes/post-list.html       Liste des articles d'une langue (utilisée par les deux accueils)
_includes/lang-switch.html     Lien automatique vers la traduction d'un article
_layouts/post.html             Gabarit d'article de minima + lien vers la traduction
assets/images/, assets/sdd/    Images et schémas (SVG), version anglaise suffixée -en.svg
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

## Site bilingue (français / anglais)

Les articles sont en français par défaut (`defaults` dans `_config.yml`).
Les deux versions d'un même article sont reliées par une clé `ref` identique ;
le lien « Read this article in English » / « Lire cet article en français »
s'affiche alors tout seul sous le titre. Sans traduction, aucun lien n'apparaît.

Article français :

```yaml
---
layout: post
title: "Mon titre"
date: 2026-11-02
ref: mon-sujet
---
```

Sa traduction, dans `_posts/2026-11-02-my-title.md` :

```yaml
---
layout: post
title: "My title"
date: 2026-11-02
ref: mon-sujet
lang: en
permalink: /en/:year/:month/:title/
---
```

- Chaque article n'apparaît que sur l'accueil de sa langue (`/` ou `/en/`).
- Pour les schémas, créez une copie anglaise suffixée `-en.svg` et pointez dessus depuis l'article anglais.
- Quand vous corrigez un article, corrigez les deux versions dans le même commit.

## Prévisualiser en local (facultatif)

Avec Ruby installé :

```bash
bundle install
bundle exec jekyll serve
```

Puis ouvrez http://localhost:4000.
