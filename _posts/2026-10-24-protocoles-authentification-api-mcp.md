---
layout: post
title: "Les protocoles d'authentification pour les applications : API REST, serveurs MCP et au-delà"
date: 2026-10-24
ref: auth-api-mcp
description: "Basic Auth, clés d'API, JWT, OAuth 2.1, OpenID Connect, SAML, mTLS, DPoP et le profil d'autorisation des serveurs MCP : comment choisir."
tags: [sécurité, authentification, oauth, api, mcp]
---

## Introduction

Il n'existe pas un protocole d'authentification idéal, mais un protocole adapté à chaque situation : qui appelle (un humain, un service, un agent IA), au nom de qui, et avec quel niveau de risque. Une clé d'API suffit pour un script interne ; une application grand public exige OAuth 2.0 et OpenID Connect ; un serveur MCP distant suit désormais un profil OAuth 2.1 précis.

Deux notions sont souvent confondues :

- **Authentification (AuthN)** : prouver *qui* l'on est (un utilisateur, une application, une machine).
- **Autorisation (AuthZ)** : déterminer *ce que* l'on a le droit de faire (lire, écrire, administrer).

OAuth 2.0 est un protocole d'**autorisation** déléguée ; OpenID Connect y ajoute la couche d'**authentification**. Cette distinction explique pourquoi les deux sont presque toujours utilisés ensemble.

Cet article passe en revue les méthodes, de la plus simple à la plus robuste, puis détaille le cas des serveurs MCP (Model Context Protocol), qui connectent les agents IA aux outils et données.

<!--more-->

## Méthodes simples : Basic Auth et clés d'API

Ces deux méthodes reposent sur un secret statique envoyé à chaque requête : faciles à mettre en œuvre, elles conviennent aux échanges internes ou de serveur à serveur, jamais à une application qui agit au nom d'utilisateurs tiers.

### HTTP Basic Authentication

Le client envoie l'identifiant et le mot de passe encodés en Base64 dans l'en-tête `Authorization` :

```http
GET /api/commandes HTTP/1.1
Authorization: Basic YWxpY2U6czNjcjN0
```

- **Avantages** : standard (RFC 7617), supporté partout, aucune infrastructure.
- **Limites** : Base64 n'est pas un chiffrement, donc HTTPS est obligatoire ; le mot de passe transite à chaque appel ; pas d'expiration, pas de portée (scope), révocation = changement de mot de passe.
- **Usage raisonnable** : outils internes, endpoints d'administration derrière un VPN, échange initial pour obtenir un jeton.

### Clés d'API (API keys)

Une chaîne aléatoire longue identifie l'application appelante, transmise dans un en-tête dédié :

```http
GET /v1/meteo?ville=Lyon HTTP/1.1
X-API-Key: sk_live_4f9a…
```

- **Avantages** : simple à émettre, révoquer et faire tourner ; idéal pour la facturation et les quotas par client.
- **Limites** : la clé identifie une *application*, pas un *utilisateur* ; c'est un secret « porteur » (quiconque la détient l'utilise) ; souvent sans expiration ni portée fine.
- **Bonnes pratiques** : la passer en en-tête, jamais dans l'URL (elle finirait dans les journaux) ; stocker côté serveur uniquement un hachage ; préfixer les clés (`sk_live_`, `sk_test_`) pour faciliter la détection de fuites ; prévoir plusieurs clés actives pour permettre la rotation sans coupure.

## Jetons : sessions, Bearer et JWT

Au lieu d'envoyer un secret permanent, le client échange une preuve d'identité contre un **jeton à durée de vie limitée**. C'est la base de toutes les méthodes modernes.

### Sessions et cookies

Après connexion, le serveur crée une session et renvoie un identifiant dans un cookie. C'est le modèle historique des applications web rendues côté serveur.

- **Atouts** : révocation immédiate (on supprime la session), cookies `HttpOnly`, `Secure`, `SameSite` qui protègent contre le vol par JavaScript.
- **Limites** : état côté serveur (stockage partagé nécessaire entre instances), protection CSRF requise, peu adapté aux clients mobiles ou aux API consommées par des tiers.

### Jetons Bearer

Un jeton « porteur » (RFC 6750) est envoyé dans l'en-tête `Authorization: Bearer <jeton>`. Celui qui le détient l'utilise : il doit donc être court, transmis en HTTPS et jamais journalisé. Le jeton peut être **opaque** (une chaîne que le serveur vérifie auprès de l'émetteur, par introspection — RFC 7662) ou **auto-porteur** (un JWT).

### JSON Web Token (JWT)

Un JWT (RFC 7519) est composé de trois parties encodées en Base64URL : `en-tête.charge-utile.signature`. La charge utile contient des *claims* :

```json
{
  "iss": "https://auth.exemple.fr",
  "sub": "user_8241",
  "aud": "https://api.exemple.fr",
  "exp": 1793011200,
  "scope": "commandes:lire"
}
```

L'API vérifie la signature avec la clé publique de l'émetteur (publiée en JWKS), puis contrôle `iss`, `aud` et `exp`, sans appel réseau.

- **Atouts** : vérification locale et rapide, idéale pour les microservices.
- **Pièges** : un JWT ne se révoque pas avant son expiration (d'où des durées courtes, 5 à 15 minutes, et des refresh tokens) ; refuser l'algorithme `none` et fixer la liste des algorithmes acceptés (RS256, ES256…) ; la charge utile est lisible par tous, elle n'est pas chiffrée ; ne jamais oublier de vérifier l'audience `aud`.

Le JWT est un **format** de jeton, pas un protocole : la question de *comment* obtenir ce jeton est réglée par OAuth 2.0.

## OAuth 2.0 et OAuth 2.1 : l'autorisation déléguée

OAuth 2.0 (RFC 6749) permet à une application d'accéder à une API **au nom d'un utilisateur, sans jamais connaître son mot de passe**. C'est le standard de fait des API publiques (Google, GitHub, Microsoft, Salesforce…). OAuth 2.1, en cours de finalisation à l'IETF, consolide une décennie de bonnes pratiques : PKCE obligatoire, suppression des flux implicite et « mot de passe ».

### Les quatre rôles

| Rôle | Description | Exemple |
| --- | --- | --- |
| Propriétaire de la ressource | L'utilisateur qui consent | Alice |
| Client | L'application qui demande l'accès | Une app mobile, un client MCP |
| Serveur d'autorisation | Authentifie et délivre les jetons | Keycloak, Auth0, Entra ID, Okta |
| Serveur de ressources | L'API protégée qui valide les jetons | L'API REST, le serveur MCP |

### Le flux recommandé : Authorization Code + PKCE

1. Le client génère un secret aléatoire (`code_verifier`) et son empreinte SHA-256 (`code_challenge`).
2. Il redirige l'utilisateur vers le serveur d'autorisation avec `client_id`, `redirect_uri`, `scope`, `state` et `code_challenge`.
3. L'utilisateur s'authentifie et donne son consentement.
4. Le serveur d'autorisation redirige vers le client avec un **code d'autorisation** à usage unique.
5. Le client échange ce code, accompagné du `code_verifier`, contre un **access token** (et souvent un **refresh token**).
6. Le client appelle l'API avec `Authorization: Bearer <access_token>`.

PKCE (RFC 7636) empêche un attaquant qui intercepterait le code de l'échanger, puisqu'il ne connaît pas le `code_verifier`. Il est désormais exigé pour tous les clients, y compris les applications serveur.

### Les autres flux (grant types)

| Flux | Cas d'usage | Utilisateur présent ? |
| --- | --- | --- |
| Authorization Code + PKCE | Applications web, mobiles, SPA, clients MCP | Oui |
| Client Credentials | Service à service, tâches planifiées, back-ends | Non |
| Device Authorization (RFC 8628) | TV, CLI, objets connectés sans navigateur | Oui, sur un autre appareil |
| Refresh Token | Renouveler un access token expiré sans nouvelle connexion | Non |
| Token Exchange (RFC 8693) | Un service échange un jeton reçu contre un jeton pour une API en aval | Non |
| ~~Implicit~~, ~~Password~~ | Supprimés dans OAuth 2.1 | — |

### Scopes, audiences et refresh tokens

- Les **scopes** (`commandes:lire`, `profil`) limitent ce que le jeton autorise ; demander le strict minimum.
- Le paramètre **resource** (RFC 8707) lie le jeton à une API précise : un jeton émis pour l'API A doit être refusé par l'API B.
- Les **refresh tokens** doivent être soumis à rotation (chaque usage en délivre un nouveau et invalide l'ancien) pour les clients publics.

## OpenID Connect et SAML : identité fédérée et SSO

Quand l'application doit savoir **qui est l'utilisateur** (connexion unique, « Se connecter avec Google »), on ajoute une couche d'identité au-dessus d'OAuth : OpenID Connect pour les applications modernes, SAML pour l'existant d'entreprise.

### OpenID Connect (OIDC)

OIDC réutilise le flux Authorization Code d'OAuth 2.0 en ajoutant le scope `openid`. Le serveur d'autorisation (appelé *OpenID Provider*) renvoie, en plus de l'access token, un **ID token** : un JWT signé qui décrit l'utilisateur (`sub`, `email`, `name`, `auth_time`…).

- **Découverte automatique** : `/.well-known/openid-configuration` publie les endpoints, les algorithmes et la clé publique (JWKS).
- **Endpoint UserInfo** pour obtenir des attributs complémentaires.
- **Règle d'or** : l'ID token sert au *client* pour connaître l'utilisateur ; l'access token sert à appeler l'*API*. Ne jamais envoyer un ID token à une API comme jeton d'accès.

### SAML 2.0

SAML (Security Assertion Markup Language) échange des **assertions XML signées** entre un fournisseur d'identité (IdP : ADFS, Okta, Entra ID) et un fournisseur de services (SP), via des redirections du navigateur.

- **Atouts** : très répandu dans les grands comptes et les SaaS B2B, mature.
- **Limites** : verbeux, complexe à implémenter correctement (attaques par contournement de signature XML), pensé pour le navigateur et non pour les API ou les applications mobiles.
- **Tendance** : les nouveaux projets privilégient OIDC ; SAML reste nécessaire pour s'intégrer aux annuaires d'entreprise existants, souvent via un courtier d'identité (Keycloak, Auth0) qui traduit SAML en OIDC.

### Et les utilisateurs eux-mêmes ?

OIDC et SAML délèguent la vérification de l'identité au fournisseur d'identité, qui peut alors utiliser l'authentification multifacteur (MFA) ou les **passkeys** (WebAuthn/FIDO2), aujourd'hui la méthode la plus résistante à l'hameçonnage.

## mTLS, signatures HMAC et DPoP : l'authentification forte entre machines

Ces mécanismes répondent à la principale faiblesse des jetons Bearer — un jeton volé est utilisable par n'importe qui — en exigeant une **preuve de possession** d'une clé que seul le client légitime détient.

### TLS mutuel (mTLS)

Dans un échange TLS classique, seul le serveur présente un certificat. En mTLS, le **client présente aussi le sien**, signé par une autorité de certification reconnue par le serveur.

- **Usages** : communications entre microservices (souvent automatisées par un *service mesh* comme Istio ou Linkerd), API bancaires et Open Banking, architectures zero trust.
- **Avec OAuth** : la RFC 8705 lie un access token au certificat du client (*certificate-bound token*) ; un jeton volé est inutilisable sans la clé privée.
- **Contraintes** : gestion du cycle de vie des certificats (émission, rotation, révocation), plus délicate à déployer côté navigateur ou mobile.

### Signature de requêtes (HMAC)

Le client et le serveur partagent une clé secrète. Le client calcule une signature (HMAC-SHA256) sur des éléments de la requête — méthode, chemin, horodatage, empreinte du corps — et l'envoie en en-tête. Le serveur recalcule et compare.

- **Exemples** : AWS Signature Version 4, vérification des webhooks (Stripe, GitHub, Slack).
- **Atouts** : le secret ne transite jamais ; l'intégrité du message est garantie ; l'horodatage bloque le rejeu.
- **Standardisation** : la RFC 9421 (HTTP Message Signatures) normalise désormais cette approche, y compris avec des clés asymétriques.

### DPoP (Demonstrating Proof of Possession)

DPoP (RFC 9449) apporte la preuve de possession au niveau applicatif, sans infrastructure de certificats. Le client génère une paire de clés et joint à chaque requête un petit JWT signé (en-tête `DPoP`) qui couvre la méthode, l'URL et l'horodatage. Le serveur d'autorisation lie l'access token à cette clé publique.

DPoP convient bien aux SPA et applications mobiles, là où mTLS est difficile à déployer.

## Le cas des serveurs MCP

Un serveur MCP distant est un **serveur de ressources OAuth 2.1** : selon la révision 2026-07-28 de la spécification, le client découvre seul le serveur d'autorisation, s'identifie par une URL (Client ID Metadata Document) et obtient un jeton valable pour ce seul serveur. Un serveur MCP local (stdio), lui, lit simplement ses identifiants dans l'environnement.

### Rappel : qu'est-ce que MCP ?

Le Model Context Protocol est un protocole ouvert qui connecte les applications d'IA (Claude, ChatGPT, VS Code, Cursor…) à des outils et des sources de données via des « serveurs MCP ». La particularité du point de vue de la sécurité : un agent peut se connecter à un serveur qu'il n'a jamais vu, sans relation préalable entre l'éditeur du client et celui du serveur. Le profil d'authentification est conçu pour ce scénario.

### Deux transports, deux approches

La spécification rend l'autorisation **optionnelle**, mais fixe la marche à suivre selon le transport.

| Transport | Où tourne le serveur | Authentification |
| --- | --- | --- |
| stdio | Processus local lancé par l'application hôte | Ne pas suivre la spécification OAuth ; identifiants lus dans l'environnement (variables, fichiers de configuration) |
| Streamable HTTP | Serveur distant accessible par URL | Profil OAuth 2.1 de la spécification MCP |

L'ancien transport HTTP+SSE (révision 2024-11-05) est déprécié.

### Le flux d'autorisation pas à pas

![Flux d'autorisation MCP : 10 échanges entre le client MCP, le serveur MCP et le serveur d'autorisation]({{ '/assets/images/flux-autorisation-mcp.svg' | relative_url }})

Le client n'a besoin que de l'URL du serveur MCP : un premier refus `401` lui indique où trouver les métadonnées, puis il enchaîne un flux Authorization Code + PKCE classique et rejoue sa requête avec le jeton.

La réponse `401` du serveur ressemble à ceci :

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.exemple.fr/.well-known/oauth-protected-resource"
```

### Les briques standards mobilisées

| Brique | Référence | Rôle dans MCP | Exigence |
| --- | --- | --- | --- |
| Protected Resource Metadata | RFC 9728 | Le serveur MCP indique quel serveur d'autorisation utiliser | Obligatoire (serveur et client) |
| Métadonnées du serveur d'autorisation | RFC 8414 et OpenID Connect Discovery | Découverte des endpoints | Le client doit gérer les deux |
| PKCE | RFC 7636 | Protège l'échange du code | Obligatoire |
| Resource Indicators | RFC 8707 | Lie le jeton au serveur MCP visé | Obligatoire |
| Client ID Metadata Documents (CIMD) | Brouillon IETF | Le `client_id` est une URL HTTPS pointant vers un JSON décrivant le client | Recommandé |
| Dynamic Client Registration | RFC 7591 | Enregistrement automatique du client | Facultatif, **déprécié** en 2026-07-28 |
| Validation de l'émetteur | RFC 9207 | Le client vérifie `iss` avant d'échanger le code | Obligatoire depuis 2026-07-28 |

### Enregistrement du client : l'ordre de priorité

Comment un client MCP obtient-il un `client_id` auprès d'un serveur d'autorisation qu'il ne connaît pas ? La spécification recommande cet ordre :

1. Utiliser des identifiants **pré-enregistrés** pour ce serveur, s'il en dispose.
2. Utiliser un **Client ID Metadata Document** si le serveur d'autorisation annonce `client_id_metadata_document_supported`.
3. Se rabattre sur le **Dynamic Client Registration** si le serveur expose un `registration_endpoint`.
4. À défaut, demander à l'utilisateur de saisir lui-même les informations du client.

Le CIMD a l'avantage d'être portable : l'identifiant est une URL hébergée par l'éditeur du client, résolue à la demande par n'importe quel serveur d'autorisation, sans base d'enregistrements à maintenir.

```json
{
  "client_id": "https://app.exemple.fr/oauth/client-metadata.json",
  "client_name": "Mon client MCP",
  "redirect_uris": ["http://127.0.0.1:3000/callback"],
  "grant_types": ["authorization_code"],
  "token_endpoint_auth_method": "none"
}
```

### Points de sécurité propres à MCP

- **Pas de transmission de jeton** : le serveur MCP ne doit jamais relayer vers une API en aval le jeton reçu du client ; il obtient son propre jeton (par exemple via Token Exchange).
- **Validation de l'audience** : le serveur MCP n'accepte que des jetons émis spécifiquement pour lui.
- **Identifiants liés à l'émetteur** : un client doit associer ses identifiants au serveur d'autorisation qui les a émis et ne jamais les réutiliser auprès d'un autre.
- **Consentement incrémental** : depuis la révision 2025-11-25, un client peut demander des scopes supplémentaires au moment où un outil en a besoin, plutôt que tout demander d'emblée.
- **Serveurs stdio** : les secrets dans l'environnement doivent avoir une portée minimale, car le processus local s'exécute avec les droits de l'utilisateur.

## Comparatif et guide de choix

Le bon choix dépend d'abord de la présence ou non d'un utilisateur humain, puis du niveau de risque.

| Méthode | Identifie | Expiration / révocation | Complexité | Cas d'usage typique |
| --- | --- | --- | --- | --- |
| Basic Auth | Un utilisateur ou un compte technique | Aucune / changement de mot de passe | Très faible | Outils internes, administration |
| Clé d'API | Une application | Manuelle / révocation simple | Faible | API publiques à quotas, intégrations serveur |
| Session + cookie | Un utilisateur | Configurable / immédiate | Faible | Applications web classiques |
| JWT | Utilisateur ou application | Courte / difficile avant expiration | Moyenne | Microservices, API sans état |
| OAuth 2.0 / 2.1 | Application agissant pour un utilisateur | Courte + refresh / via le serveur d'autorisation | Moyenne à élevée | API ouvertes à des tiers, clients MCP distants |
| OpenID Connect | Un utilisateur (identité) | Courte | Moyenne | SSO, « Se connecter avec… » |
| SAML 2.0 | Un utilisateur (identité) | Courte | Élevée | SSO d'entreprise historique |
| mTLS | Une machine | Durée du certificat / CRL, OCSP | Élevée | Zero trust, service mesh, finance |
| Signature HMAC | Une application + intégrité du message | Par requête (horodatage) | Moyenne | Webhooks, API cloud |
| DPoP | Application, jeton lié à une clé | Courte | Moyenne | SPA et mobiles à haut niveau de sécurité |

**Raccourci de décision :**

- Un utilisateur se connecte à *votre* application → **OIDC** (avec passkeys ou MFA chez le fournisseur d'identité).
- Une application tierce accède aux données d'un utilisateur → **OAuth 2.1 Authorization Code + PKCE**.
- Un service appelle un autre service → **Client Credentials**, renforcé par **mTLS** ou **DPoP** si les données sont sensibles.
- Vous recevez des webhooks → **signature HMAC**.
- Un script interne ou un prototype → **clé d'API**, avec rotation.
- Un serveur MCP distant → **le profil OAuth 2.1 de la spécification MCP** (section précédente) ; un serveur MCP local → **variables d'environnement**.

## Bonnes pratiques et conclusion

Quel que soit le protocole retenu, la plupart des incidents viennent de détails d'implémentation plutôt que du protocole lui-même.

- **HTTPS partout**, sans exception, y compris en interne.
- **Ne pas réinventer** : s'appuyer sur un serveur d'autorisation éprouvé (Keycloak, Auth0, Entra ID, Okta, Cognito…) et des bibliothèques maintenues.
- **Jetons courts** (minutes) et refresh tokens soumis à rotation.
- **Valider chaque jeton** : signature, émetteur `iss`, audience `aud`, expiration `exp`, scopes.
- **Moindre privilège** : scopes minimaux, un jeton par API cible.
- **Secrets hors du code** : coffre-fort (Vault, AWS Secrets Manager…) ou variables d'environnement, jamais dans Git.
- **Rien de sensible dans les URL ni dans les journaux** (clés, jetons, codes d'autorisation).
- **Limitation de débit** sur les endpoints d'authentification pour freiner le bourrage d'identifiants.
- **Journaliser et surveiller** les émissions de jetons, les échecs et les révocations.
- **Pour les agents IA** : consentement explicite de l'utilisateur, jetons limités au serveur MCP visé, pas de transmission de jeton vers les API en aval.

En résumé, les clés d'API et Basic Auth restent utiles pour les cas simples ; OAuth 2.1 et OpenID Connect sont la norme dès qu'un utilisateur et des tiers entrent en jeu ; mTLS, HMAC et DPoP durcissent les échanges entre machines. Les serveurs MCP ne réinventent rien : ils appliquent ces standards avec un profil strict, conçu pour que des agents IA puissent se connecter à des milliers de services sans enregistrement préalable.

## Sources

- [MCP — Authorization, révision 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [MCP — Client Registration](https://modelcontextprotocol.io/specification/latest/basic/authorization/client-registration)
- [MCP Authorization Spec 2026-07-28 : ce qui change (SSOJet)](https://ssojet.com/blog/mcp-authorization-spec-2026-07-28-what-changed)
- [MCP Authentication : exigences de la révision 2026-07-28 (SSOJet)](https://ssojet.com/blog/mcp-authentication-oauth-tokens-security-ai-connections.md)
- [Nouveautés de la spécification d'autorisation MCP 2025-11-25 (den.dev)](https://den.dev/blog/mcp-november-authorization-spec/)
- RFC citées : 6749 (OAuth 2.0), 6750 (Bearer), 7519 (JWT), 7617 (Basic), 7636 (PKCE), 7662 (introspection), 8414, 8628, 8693, 8705, 8707, 9207, 9421, 9449, 9728 — consultables sur [rfc-editor.org](https://www.rfc-editor.org)
