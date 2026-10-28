---
layout: post
title: "Du besoin métier au frontend : maquettes et code générés par l'IA avec le Specification-Driven Design"
date: 2026-10-28
ref: sdd
categories: [ia, design, frontend]
tags:
  [
    SDD,
    spec-driven,
    IA,
    maquettes,
    design system,
    design tokens,
    frontend,
    agents,
  ]
---

# Du besoin métier au frontend : maquettes et code générés par l'IA avec le Specification-Driven Design

« Il nous faudrait un écran pour que les collaborateurs posent leurs congés et que les managers les valident. »

Il y a quelques années, cette phrase déclenchait des semaines d'ateliers, de wireframes, d'allers-retours avec les designers puis de développement. Aujourd'hui, un agent IA peut produire en quelques minutes des maquettes crédibles et le code React qui va avec. Le problème n'est plus de _produire_ : c'est de produire **la bonne chose**, de façon **cohérente**, **reproductible** et **maîtrisée**.

C'est exactement ce que propose le **Specification-Driven Design** (ou _Spec-Driven Development_, SDD) : faire de la spécification, et non du prompt ou du code, la source de vérité de toute la chaîne. Cet article montre comment l'appliquer de bout en bout, du besoin métier jusqu'au frontend, avec une attention particulière aux bénéfices réels et aux enjeux qu'il ne faut pas ignorer.

---

## Sommaire

1. [Le problème du « vibe coding »](#1-le-problème-du--vibe-coding-)
2. [Le principe du Specification-Driven Design](#2-le-principe-du-specification-driven-design)
3. [Vue d'ensemble du pipeline](#3-vue-densemble-du-pipeline)
4. [Mode opératoire étape par étape](#4-mode-opératoire-étape-par-étape)
5. [Organisation du dépôt](#5-organisation-du-dépôt)
6. [L'outillage disponible](#6-loutillage-disponible)
7. [Les avantages](#7-les-avantages)
8. [Avoir conscience des enjeux](#8-avoir-conscience-des-enjeux)
9. [Où placer l'humain : la grille de contrôle](#9-où-placer-lhumain--la-grille-de-contrôle)
10. [Par où commencer](#10-par-où-commencer)
11. [Conclusion](#11-conclusion)

---

## 1. Le problème du « vibe coding »

La manière la plus intuitive d'utiliser l'IA pour créer une interface consiste à décrire ce qu'on veut dans un chat, regarder le résultat, puis corriger par petites touches : « mets le bouton à droite », « non, plus clair », « ajoute un filtre ». C'est ce qu'on appelle familièrement le _vibe coding_.

Ça fonctionne très bien pour un prototype jetable. Ça fonctionne beaucoup moins bien dès qu'on vise un produit réel :

- **L'intention se perd.** Les décisions sont dispersées dans un historique de conversation que personne ne relira. Pourquoi ce champ est-il obligatoire ? Personne ne s'en souvient.
- **Le résultat n'est pas reproductible.** Le même prompt, relancé demain ou avec un autre modèle, produit une interface différente.
- **La cohérence se dégrade.** Chaque écran est généré isolément ; les couleurs, les espacements et les comportements divergent.
- **La revue est difficile.** On ne peut pas valider un besoin métier en relisant 2 000 lignes de JSX.
- **Les cas limites sont oubliés.** L'IA dessine volontiers le cas nominal ; les états vides, d'erreur ou de chargement, les droits d'accès, l'accessibilité passent à la trappe.

Le Specification-Driven Design répond à ces cinq problèmes par une idée simple : **on n'écrit plus de prompts, on écrit des spécifications.**

---

## 2. Le principe du Specification-Driven Design

Le SDD repose sur trois convictions.

**1. La spécification est la source de vérité.** Les maquettes et le code sont des _artefacts dérivés_, régénérables à partir de la spec. Si le résultat ne convient pas, on ne retouche pas le code à la main : on corrige la spec et on régénère.

**2. La spécification est structurée et progressive.** On ne passe pas d'une phrase métier à du code en une étape. On raffine par couches successives, chacune lisible et validable par la bonne personne :

| Couche             | Question à laquelle elle répond      | Relue par             |
| ------------------ | ------------------------------------ | --------------------- |
| Besoin métier      | Pourquoi ? Pour qui ?                | Métier, product owner |
| Spec fonctionnelle | Quoi ? Selon quelles règles ?        | Product owner, métier |
| Spec d'expérience  | Comment l'utilisateur s'y prend-il ? | UX designer           |
| Design system      | À quoi cela ressemble-t-il ?         | UI designer           |
| Plan technique     | Avec quelle architecture ?           | Tech lead             |

**3. La spécification vit dans le dépôt Git.** Ce sont des fichiers Markdown, JSON ou YAML, versionnés à côté du code, soumis aux mêmes pull requests et à la même revue. L'historique Git devient l'historique des décisions.

Ce mouvement s'est structuré en 2025 avec l'apparition d'outils dédiés (GitHub Spec Kit, Kiro d'AWS, OpenSpec, BMAD-METHOD, Tessl…) qui proposent tous, avec des variantes, la même séquence : **spécifier → clarifier → planifier → découper en tâches → implémenter**.

---

## 3. Vue d'ensemble du pipeline

![Pipeline spec-driven : du besoin métier (1) à la spec fonctionnelle (2), la spec d'expérience (3), le design system (4), les maquettes générées (5), le plan technique (6), le code frontend (7) et la vérification (8), avec des validations humaines et une boucle de retour vers la spec]({{ '/assets/sdd/pipeline-sdd.svg' | relative_url }})

Trois familles d'éléments se distinguent :

- **En bleu, les spécifications** : écrites ou co-écrites avec l'IA, validées par un humain, versionnées. Elles font foi.
- **En violet, les artefacts générés** : maquettes et code, produits par l'IA à partir des specs. Ils sont jetables et régénérables.
- **En vert, les contrôles automatiques** : tests, audit d'accessibilité, comparaison visuelle.

Les losanges orange marquent les **points de validation humaine**. La flèche en pointillés est la règle d'or : quand la vérification révèle un écart, **on remonte à la spec**.

---

## 4. Mode opératoire étape par étape

Fil rouge : une application interne de **demande de congés**.

### Étape 0 — Poser la « constitution » du projet

Avant toute fonctionnalité, on écrit une fois pour toutes les règles non négociables que l'IA devra respecter à chaque génération. Spec Kit appelle ce fichier la _constitution_ ; d'autres outils parlent de _steering_ ou de règles projet.

```markdown
<!-- specs/constitution.md -->

# Constitution du projet

## Stack

- React 19 + TypeScript strict, Vite, React Router.
- Composants UI : uniquement ceux de `src/design-system/`. Aucun style en dur.
- Appels API : uniquement via le client généré depuis `contracts/openapi.yaml`.

## Qualité

- Accessibilité : WCAG 2.2 niveau AA obligatoire.
- Chaque écran gère explicitement les états : chargement, vide, erreur, succès.
- Chaque critère d'acceptation de la spec a au moins un test end-to-end.

## Interdits

- Pas de nouvelle dépendance npm sans mention dans le plan et validation humaine.
- Pas de donnée personnelle réelle dans les fixtures et les maquettes.
- Pas de secret ni de clé d'API dans le code.
```

Ce fichier est l'outil le plus rentable de toute la démarche : il évite de répéter les mêmes consignes dans chaque prompt, et il rend les écarts détectables.

### Étape 1 — Exprimer le besoin métier

Le point de départ reste un texte en langage naturel, écrit par le métier. Pas besoin qu'il soit parfait ; il doit surtout dire **pourquoi**.

```markdown
<!-- specs/001-conges/besoin.md -->

# Besoin : gestion des demandes de congés

Aujourd'hui, les demandes de congés passent par e-mail et un tableur partagé.
Les managers perdent des demandes, les collaborateurs ne connaissent pas leur solde,
et les RH ressaisissent tout en fin de mois.

Nous voulons que :

- un collaborateur puisse poser une demande et voir son solde en temps réel ;
- son manager soit notifié et puisse valider ou refuser en un clic ;
- les RH puissent exporter les congés validés du mois.

Contraintes : utilisable sur mobile, conforme à nos règles d'accessibilité.
Indicateur de succès : moins de 2 minutes pour poser une demande.
```

### Étape 2 — Générer puis clarifier la spec fonctionnelle

On demande à l'agent de transformer le besoin en spécification structurée : rôles, user stories, règles métier, critères d'acceptation testables. Une notation contrainte aide énormément ; Kiro utilise par exemple la notation **EARS** (_Easy Approach to Requirements Syntax_), qui impose des formes comme « QUAND… LE SYSTÈME DOIT… ».

```markdown
<!-- specs/001-conges/spec.md (extrait) -->

## Rôles

- Collaborateur, Manager, RH.

## US-01 — Poser une demande

En tant que collaborateur, je veux poser une demande de congés
afin de planifier mon absence sans passer par e-mail.

### Critères d'acceptation

- AC-01.1 QUAND le collaborateur ouvre le formulaire,
  LE SYSTÈME DOIT afficher son solde disponible par type de congé.
- AC-01.2 SI la date de fin est antérieure à la date de début,
  ALORS LE SYSTÈME DOIT bloquer l'envoi et afficher une erreur sur le champ concerné.
- AC-01.3 SI le nombre de jours demandés dépasse le solde,
  ALORS LE SYSTÈME DOIT avertir sans bloquer l'envoi.
- AC-01.4 QUAND la demande est envoyée,
  LE SYSTÈME DOIT afficher une confirmation et notifier le manager.

## Questions ouvertes

- [ ] Les demi-journées sont-elles autorisées ?
- [ ] Un manager peut-il valider sa propre demande ?
```

La section **« Questions ouvertes »** est essentielle. Plutôt que de laisser l'IA trancher en silence, on lui demande explicitement de lister ses ambiguïtés : c'est l'étape _clarify_ de Spec Kit. Le métier y répond, la spec est mise à jour, puis **validée** (première porte humaine).

### Étape 3 — Spécifier l'expérience

Cette couche décrit les parcours et les écrans **sans parler encore d'apparence** : quels écrans, quelles informations, quelles actions, quels états.

```markdown
<!-- specs/001-conges/ux.md (extrait) -->

## Parcours P1 — Poser une demande (collaborateur, mobile d'abord)

Accueil → « Nouvelle demande » → Formulaire → Récapitulatif → Confirmation

## Écran E2 — Formulaire de demande

Contenu :

- Carte « Mon solde » (par type de congé) → AC-01.1
- Type de congé (liste), date de début, date de fin, demi-journée (si validé)
- Commentaire facultatif (500 caractères max)
  Actions : « Vérifier ma demande » (principale), « Annuler » (secondaire)

États :

- Chargement : squelette de la carte solde, formulaire désactivé
- Erreur de chargement du solde : message + bouton « Réessayer »
- Erreur de validation : message sous le champ, focus déplacé dessus → AC-01.2
- Dépassement de solde : bandeau d'avertissement non bloquant → AC-01.3
```

Remarquez la **traçabilité** : chaque élément d'écran renvoie au critère d'acceptation qui le justifie. C'est ce qui permettra plus tard de vérifier automatiquement que rien n'a été oublié.

### Étape 4 — Formaliser le design system

Pour que l'IA ne réinvente pas un style à chaque écran, l'identité visuelle doit être **elle aussi une spécification**. Le format standard est celui des **design tokens** du W3C : la _Design Tokens Community Group_ a publié en octobre 2025 la première version stable de sa spécification (2025.10), déjà prise en charge par Figma, Penpot, Sketch, Style Dictionary, Tokens Studio ou Terrazzo.

```json
{
  "color": {
    "brand": {
      "primary": {
        "$type": "color",
        "$value": {
          "colorSpace": "srgb",
          "components": [0.145, 0.388, 0.922],
          "hex": "#2563eb"
        }
      }
    },
    "feedback": {
      "warning": {
        "$type": "color",
        "$value": {
          "colorSpace": "srgb",
          "components": [0.706, 0.325, 0.035],
          "hex": "#b45309"
        }
      }
    }
  },
  "space": {
    "md": { "$type": "dimension", "$value": { "value": 16, "unit": "px" } }
  }
}
```

On y ajoute le **catalogue des composants** autorisés (bouton, champ, carte, bandeau…) avec leurs variantes et leurs règles d'usage. Avantage majeur : un même fichier de tokens peut générer les variables CSS du web, les ressources iOS et Android, et alimenter l'outil de maquettage.

Si l'entreprise a déjà un design system, on le reprend tel quel : c'est même là que l'approche est la plus efficace.

### Étape 5 — Générer les maquettes

On dispose maintenant de tout ce qu'il faut pour générer des maquettes **contraintes** : l'agent reçoit la spec d'expérience (quels écrans, quels états) et le design system (quels composants, quels tokens). Il produit :

- un écran par entrée de `ux.md`, **avec chacun de ses états** (chargement, vide, erreur, succès) ;
- en version mobile et desktop ;
- en utilisant exclusivement les composants du catalogue.

Selon les outils, les maquettes sont produites directement dans un outil de design (Figma Make s'appuie par exemple sur les bibliothèques de composants existantes de l'équipe), dans un outil de génération d'interfaces (Google Stitch, Uizard, Visily, UX Pilot), ou sous forme de prototype HTML. Grâce à MCP, un agent de code peut aussi lire et écrire dans l'outil de design.

C'est la deuxième porte humaine, et sans doute la plus importante : **le designer et le métier valident les maquettes**. Une remarque (« le solde devrait être visible avant même d'ouvrir le formulaire ») ne se corrige pas dans la maquette : elle se traduit en modification de `ux.md`, puis on régénère.

### Étape 6 — Planifier l'implémentation

L'agent produit ensuite un **plan technique** qui respecte la constitution, puis une **liste de tâches** ordonnées et de petite taille.

```markdown
<!-- specs/001-conges/tasks.md (extrait) -->

- [ ] T01 Générer le client API depuis contracts/openapi.yaml (endpoints /leaves, /balances)
- [ ] T02 Créer la route /conges/nouvelle et le squelette de page (E2)
- [ ] T03 Composant LeaveBalanceCard : états chargement / erreur / succès (AC-01.1)
- [ ] T04 Formulaire : validation des dates, message d'erreur, gestion du focus (AC-01.2)
- [ ] T05 Bandeau d'avertissement de dépassement de solde (AC-01.3)
- [ ] T06 Tests Playwright AC-01.1 → AC-01.4 + audit axe-core sur E2
```

Le **contrat d'API** (OpenAPI) mérite une mention : c'est une spécification à part entière, partagée avec l'équipe backend. Le frontend peut être développé et testé contre un serveur simulé généré depuis ce contrat, avant même que l'API existe.

### Étape 7 — Générer le code

L'agent de code (Claude Code, Copilot, Cursor, Codex, Gemini CLI, Kiro…) exécute les tâches une par une. Chaque tâche est petite, référencée et vérifiable, ce qui limite fortement les dérives.

On garde une **revue de code humaine** sur les pull requests : elle ne porte plus sur la mise en forme, mais sur ce que l'IA rate le plus souvent (sécurité, gestion des erreurs, dépendances ajoutées, performances).

### Étape 8 — Vérifier automatiquement

La vérification ferme la boucle. Comme chaque critère d'acceptation est formulé de façon testable, il se traduit directement en test :

```typescript
// e2e/conges/us-01.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("AC-01.2 — une date de fin antérieure bloque l'envoi", async ({
  page,
}) => {
  await page.goto("/conges/nouvelle");
  await page.getByLabel("Date de début").fill("2026-11-10");
  await page.getByLabel("Date de fin").fill("2026-11-05");
  await page.getByRole("button", { name: "Vérifier ma demande" }).click();

  const fin = page.getByLabel("Date de fin");
  await expect(fin).toHaveAttribute("aria-invalid", "true");
  await expect(fin).toBeFocused();
});

test("E2 — aucune violation d'accessibilité WCAG A/AA", async ({ page }) => {
  await page.goto("/conges/nouvelle");
  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21aa", "wcag22aa"])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

On complète avec :

- une **comparaison visuelle** entre les captures du frontend et les maquettes validées ;
- un **contrôle de traçabilité** : un script vérifie que chaque `AC-xx` de la spec est couvert par au moins un test ;
- une **vérification de conformité** au design system : aucune couleur ou taille codée en dur.

Quand un contrôle échoue, la règle reste la même : on identifie si l'erreur vient de la spec ou de la génération, et on corrige à la source.

---

## 5. Organisation du dépôt

```text
mon-app/
├── specs/
│   ├── constitution.md            # règles non négociables
│   ├── design-system/
│   │   ├── tokens.json            # design tokens (format DTCG)
│   │   └── composants.md          # catalogue et règles d'usage
│   └── 001-conges/
│       ├── besoin.md              # expression métier
│       ├── spec.md                # stories + critères (validée)
│       ├── ux.md                  # parcours, écrans, états
│       ├── plan.md                # architecture
│       └── tasks.md               # tâches ordonnées
├── contracts/
│   └── openapi.yaml               # contrat d'API
├── maquettes/                     # artefacts générés (ou lien vers l'outil de design)
├── src/                           # code généré et revu
└── e2e/                           # tests dérivés des critères d'acceptation
```

Chaque fonctionnalité a son dossier de specs numéroté ; une pull request contient à la fois la modification de la spec et le code régénéré, ce qui permet de relire **l'intention et sa réalisation** au même endroit.

---

## 6. L'outillage disponible

L'écosystème évolue très vite ; ce panorama donne des repères par catégorie plutôt qu'un classement.

| Besoin                          | Exemples d'outils                                      | Remarques                                                                                                                                                      |
| ------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cadre SDD                       | GitHub Spec Kit, Kiro, OpenSpec, BMAD-METHOD, Tessl    | Spec Kit est open source et fonctionne avec la plupart des agents ; Kiro est un IDE complet lié à l'écosystème AWS ; BMAD est plus lourd, pour de gros projets |
| Agents de code                  | Claude Code, GitHub Copilot, Cursor, Codex, Gemini CLI | Exécutent les tâches, lisent la constitution                                                                                                                   |
| Génération de maquettes         | Figma Make, Google Stitch, Uizard, Visily, UX Pilot    | Figma Make s'appuie sur vos bibliothèques ; Stitch reste un produit expérimental de Google Labs                                                                |
| Génération d'interfaces en code | v0, Lovable, Bolt                                      | Très rapides ; à encadrer pour respecter votre design system                                                                                                   |
| Design tokens                   | Style Dictionary, Tokens Studio, Terrazzo              | Implémentations de référence du format DTCG                                                                                                                    |
| Vérification                    | Playwright, axe-core, outils de régression visuelle    | Indispensables pour fermer la boucle                                                                                                                           |
| Connexion des outils            | Serveurs MCP (outil de design, ticketing, dépôt)       | Permettent à l'agent de lire et d'écrire dans vos outils                                                                                                       |

Bon réflexe : choisir des outils qui stockent les specs **en clair dans le dépôt** (Markdown, JSON). C'est ce qui limite le verrouillage chez un fournisseur.

---

## 7. Les avantages

**Une vitesse qui ne sacrifie pas la cohérence.** Le temps entre un besoin exprimé et un premier écran fonctionnel passe de semaines à quelques heures, mais contrairement au vibe coding, tous les écrans respectent le même design system et les mêmes règles.

**Des décisions explicites et traçables.** Chaque choix est écrit, relu, versionné. Six mois plus tard, on sait pourquoi un champ existe et quel critère il satisfait. C'est précieux pour l'onboarding, la maintenance et les audits.

**Une revue au bon niveau.** Le métier relit un document métier, le designer relit des maquettes, le développeur relit du code. Personne n'a à déchiffrer un artefact qui n'est pas de son ressort.

**Des cas limites traités par construction.** Parce que la spec d'expérience impose les états d'erreur, de chargement et de vide, et que la constitution impose l'accessibilité, ces sujets ne sont plus oubliés en fin de projet.

**Une indépendance vis-à-vis du modèle.** Une spec bien écrite peut être réexécutée avec un autre agent ou un modèle plus récent. On capitalise sur les specs, pas sur des prompts éphémères.

**Une meilleure collaboration.** Le besoin métier, la spec et les maquettes deviennent un langage commun entre métier, design et technique. Les maquettes générées très tôt permettent de valider avec les utilisateurs avant d'écrire du code.

**Des tests qui découlent de la spec.** Les critères d'acceptation deviennent des tests ; la couverture fonctionnelle devient mesurable.

**Une évolution facilitée.** Ajouter une règle métier revient à modifier la spec et régénérer les parties concernées, plutôt qu'à chercher où intervenir dans le code.

---

## 8. Avoir conscience des enjeux

L'automatisation de bout en bout est séduisante. Elle comporte aussi des risques, dont certains sont nouveaux. Les connaître est ce qui distingue une adoption maîtrisée d'un pari.

### « Entrée médiocre, sortie médiocre »

L'IA amplifie la qualité de la spec, dans les deux sens. Une spec ambiguë produit une interface plausible mais fausse, et d'autant plus trompeuse qu'elle a l'air finie. **Bien spécifier devient la compétence centrale**, et c'est un vrai travail. Le SDD ne supprime pas l'effort d'analyse : il le déplace en amont.

### L'illusion du « totalement automatisé »

La _production_ peut être entièrement automatisée. La _décision_ ne doit pas l'être. Une chaîne sans porte humaine validera elle-même ses propres erreurs : l'IA qui écrit la spec, génère le code et écrit les tests peut produire un ensemble parfaitement cohérent… et hors sujet. Il faut au minimum un humain qui valide la spec, un qui valide les maquettes, et une revue du code.

### Le risque de waterfall déguisé

Produire beaucoup de documents avant de coder rappelle le cycle en V, et des critiques de Spec Kit pointent justement ce poids pour de petites évolutions. La parade : des specs **courtes et incrémentales** (une fonctionnalité à la fois), et une génération si rapide qu'on peut montrer des maquettes au métier dès le premier jour. Pour un correctif d'une ligne, inutile de dérouler tout le pipeline.

### La dérive entre spec et code

Si quelqu'un corrige « juste un détail » directement dans le code, la spec cesse d'être la vérité et le prochain cycle de régénération écrasera la correction. Il faut une discipline d'équipe (on corrige la spec), éventuellement outillée (hooks qui signalent les écarts, contrôle de traçabilité en CI).

### L'accessibilité n'est pas acquise

Le code généré par IA reproduit les défauts fréquents du web : libellés manquants, contrastes insuffisants, navigation clavier cassée, rôles ARIA superflus. Les audits automatiques n'en détectent qu'une partie ; des tests avec des technologies d'assistance et, idéalement, avec des utilisateurs concernés restent nécessaires. C'est aussi un enjeu réglementaire : en Europe, l'_European Accessibility Act_ s'applique depuis juin 2025 à de nombreux services numériques grand public.

### Sécurité et chaîne d'approvisionnement

Un agent qui génère du code peut introduire une dépendance inutile, vulnérable ou même inexistante (le nom d'un paquet inventé par un modèle peut être enregistré par un attaquant). Il peut aussi manipuler des secrets, ou exposer des données côté client. Autres points de vigilance : les specs et les fichiers lus par l'agent sont une **surface d'injection de prompt** ; un contenu malveillant glissé dans une spec ou un ticket peut orienter la génération. Il faut donc relire les specs comme on relit du code, restreindre les droits de l'agent et analyser les dépendances en CI.

### Confidentialité et propriété intellectuelle

Le besoin métier, les règles de gestion et parfois des exemples de données sont envoyés à un modèle hébergé. Il faut vérifier les conditions contractuelles du fournisseur (rétention, réutilisation des données), ne jamais placer de données personnelles réelles dans les specs ou les fixtures (RGPD), et s'interroger sur les droits attachés aux maquettes et au code générés, ainsi que sur la ressemblance éventuelle avec des interfaces existantes.

### L'uniformisation esthétique

Les outils de génération ont tendance à converger vers les mêmes interfaces : mêmes bibliothèques de composants, mêmes mises en page, même esthétique « par défaut ». Pour une application interne, ce n'est pas un problème. Pour un produit qui doit se différencier, le travail de direction artistique et un design system propre restent indispensables ; l'IA décline une identité, elle ne la crée pas à votre place.

### Dette technique et maintenabilité

Le code généré peut être verbeux, dupliqué, ou s'appuyer sur des motifs différents d'une génération à l'autre. Une constitution précise (architecture, conventions, bibliothèques autorisées), des tâches petites et une revue humaine limitent ce risque. Il faut aussi se demander : si l'outil de génération disparaît demain, l'équipe sait-elle maintenir ce code à la main ?

### Coûts et empreinte

Chaque régénération consomme des ressources de calcul : coût financier (les outils facturent souvent au crédit ou au jeton) et coût environnemental. Régénérer dix fois un écran pour un détail visuel n'est pas neutre. Des specs claires réduisent le nombre d'itérations.

### L'évolution des métiers

Le SDD ne supprime pas les designers et les développeurs, mais déplace leur valeur. Le designer passe de la production d'écrans à la conception du système (design system, principes d'interaction, validation). Le développeur passe de l'écriture de composants à l'architecture, à la revue et à la qualité. Le métier et le product owner prennent une responsabilité accrue dans l'écriture de specs précises. Cette transition demande de la formation et un accompagnement, en particulier pour les profils juniors, qui apprenaient jusqu'ici en produisant justement ce que l'IA produit désormais.

### Le verrouillage

Certains outils stockent les specs dans un format propriétaire, imposent leur IDE ou leur modèle, ou découragent la modification manuelle du code généré. Privilégier des specs en Markdown/JSON dans le dépôt et des standards ouverts (design tokens DTCG, OpenAPI, MCP) garde la porte ouverte à un changement d'outil.

---

## 9. Où placer l'humain : la grille de contrôle

| Étape              | Produit par                         | Validé par                                         | Critère de passage                                          |
| ------------------ | ----------------------------------- | -------------------------------------------------- | ----------------------------------------------------------- |
| Constitution       | Tech lead + design lead (avec l'IA) | Équipe                                             | Règles connues et acceptées                                 |
| Besoin métier      | Métier                              | Product owner                                      | Le « pourquoi » et l'indicateur de succès sont clairs       |
| Spec fonctionnelle | IA, à partir du besoin              | **Métier + PO**                                    | Aucune question ouverte, critères testables                 |
| Spec d'expérience  | IA + UX designer                    | **UX designer**                                    | Tous les états décrits, traçabilité vers les critères       |
| Design system      | Designer (outils IA en appui)       | **Design lead**                                    | Tokens au format standard, contrastes conformes             |
| Maquettes          | IA                                  | **Designer + métier** (+ utilisateurs si possible) | Parcours validés, aucun composant hors catalogue            |
| Plan et tâches     | IA                                  | **Tech lead**                                      | Respect de la constitution, pas de dépendance non justifiée |
| Code               | IA                                  | **Revue de code**                                  | Sécurité, erreurs, performance                              |
| Vérification       | Automatique                         | CI                                                 | Tous les critères couverts, zéro violation a11y bloquante   |

La règle simple à retenir : **l'IA propose, un humain qualifié dispose, à chaque changement de couche.**

---

## 10. Par où commencer

1. **Choisir une fonctionnalité pilote** limitée et à faible risque (un écran interne, un formulaire).
2. **Écrire la constitution** du projet, même courte : stack, design system, accessibilité, interdits.
3. **Exporter le design system existant en tokens** au format DTCG. S'il n'existe pas, commencer par une palette, une échelle d'espacements et cinq composants.
4. **Adopter un cadre SDD léger** (Spec Kit ou une simple convention de dossiers `specs/`) avec l'agent déjà utilisé par l'équipe.
5. **Imposer la section « Questions ouvertes »** et ne rien générer tant qu'elle n'est pas vide.
6. **Brancher les vérifications dès le départ** : tests dérivés des critères, audit d'accessibilité, contrôle de traçabilité.
7. **Mesurer** : temps du besoin à la mise en production, nombre d'allers-retours, défauts détectés après livraison. Comparer avec la méthode habituelle avant de généraliser.

---

## 11. Conclusion

L'IA a rendu la production de maquettes et de code presque gratuite. Ce qui devient rare et précieux, c'est la **clarté de l'intention**. Le Specification-Driven Design en tire la conséquence logique : on investit dans la spécification, on laisse l'IA dériver les artefacts, et on place des humains aux moments où une décision se prend.

Bien mené, il permet de passer d'un besoin métier à un frontend cohérent, accessible et testé en un temps inédit. Mal mené, il automatise surtout la production d'erreurs bien présentées. La différence tient à trois choses : la qualité des specs, la discipline de corriger à la source, et des validations humaines aux bons endroits.

---

## Pour aller plus loin

- [GitHub Spec Kit vs Kiro vs Claude Code : comparaison des workflows SDD](https://glukhov.org/ai-devtools/ai-coding-assistants/spec-kit-vs-kiro-vs-claude-code/)
- [7 outils de spec-driven development (Spec Kit, Kiro, OpenSpec, Tessl…)](https://ssojet.com/blog/best-spec-driven-development-tools)
- [Spec-driven development avec Kiro et les agents IA](https://www.morphllm.com/spec-driven-development)
- [Design Tokens : première version stable de la spécification (W3C DTCG)](https://www.w3.org/community/design-tokens/2025/10/28/design-tokens-specification-reaches-first-stable-version/)
- [Panorama des outils IA pour l'UI/UX en 2026](https://resources.rework.com/tools/ai-tools/best-ai-tools-for-ui-ux-design-2026)
- Article précédent sur ce blog : le protocole A2A et sa comparaison avec MCP
