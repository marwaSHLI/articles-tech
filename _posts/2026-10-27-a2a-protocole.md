---
layout: post
title: "Le protocole A2A : fonctionnement, mode opératoire et comparaison avec MCP"
date: 2026-10-27
categories: [ia, agents, protocoles]
tags: [A2A, MCP, agents, interopérabilité, LLM]
---

# Le protocole A2A : fonctionnement, mode opératoire et comparaison avec MCP

Les systèmes d'IA ne se résument plus à un seul modèle qui répond à des questions. On construit désormais des **agents** spécialisés — un agent RH, un agent de réservation, un agent de revue de code — souvent développés par des équipes différentes, sur des frameworks différents (LangGraph, CrewAI, ADK, OpenAI Agents SDK…), et hébergés chez des fournisseurs différents. Le problème devient alors : **comment ces agents se parlent-ils ?**

C'est la question à laquelle répond **A2A (Agent2Agent)**. Cet article présente le protocole, détaille son mode opératoire pas à pas, puis le compare à **MCP (Model Context Protocol)**, avec lequel il est souvent confondu.

---

## Sommaire

1. [Qu'est-ce que A2A ?](#1-quest-ce-que-a2a-)
2. [Historique et gouvernance](#2-historique-et-gouvernance)
3. [Les concepts clés](#3-les-concepts-clés)
4. [Architecture en trois couches](#4-architecture-en-trois-couches)
5. [Mode opératoire pas à pas](#5-mode-opératoire-pas-à-pas)
6. [Cycle de vie d'une tâche](#6-cycle-de-vie-dune-tâche)
7. [Sécurité](#7-sécurité)
8. [A2A vs MCP](#8-a2a-vs-mcp)
9. [Utiliser A2A et MCP ensemble](#9-utiliser-a2a-et-mcp-ensemble)
10. [Limites actuelles et bonnes pratiques](#10-limites-actuelles-et-bonnes-pratiques)
11. [Conclusion](#11-conclusion)

---

## 1. Qu'est-ce que A2A ?

A2A est une **spécification ouverte** qui définit comment des agents IA indépendants :

- **se découvrent** (qui sait faire quoi, et comment le contacter),
- **se délèguent des tâches**, y compris longues et multi-étapes,
- **échangent des données** (texte, JSON, fichiers) de façon sécurisée,
- **suivent l'avancement** d'un travail en cours (streaming, notifications).

L'idée centrale : chaque agent est traité comme une **boîte noire**. Un agent client n'a pas besoin de connaître le modèle, le prompt, les outils ou la mémoire de l'agent distant. Il lui confie une tâche, comme on confierait un dossier à un collègue d'un autre service, et récupère un résultat.

> **En une phrase :** MCP relie un agent à ses _outils_ ; A2A relie un agent à d'_autres agents_.

---

## 2. Historique et gouvernance

| Date       | Étape                                                                                                                                            |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Avril 2025 | Annonce du protocole par Google, avec une cinquantaine de partenaires                                                                            |
| Juin 2025  | Don du projet à la **Linux Foundation**, qui en assure désormais la gouvernance neutre                                                           |
| Été 2025   | Version 0.3 : ajout d'une liaison gRPC, signature des Agent Cards                                                                                |
| Mars 2026  | **Version 1.0 stable**                                                                                                                           |
| 2026       | Support en production dans plusieurs plateformes (Microsoft Copilot Studio, Azure AI Foundry, Amazon Bedrock AgentCore, ServiceNow, Salesforce…) |

La v1.0 a introduit plusieurs changements cassants par rapport aux versions 0.x (noms de méthodes, noms d'états, suppression de certains champs). Si vous lisez des tutoriels de 2025, vérifiez la version ciblée.

---

## 3. Les concepts clés

A2A repose sur un petit vocabulaire :

| Concept                        | Rôle                                                                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Client agent**               | L'agent qui initie la demande (souvent un orchestrateur).                                                                                  |
| **Remote agent (serveur A2A)** | L'agent qui reçoit et exécute la tâche. Les rôles sont fluides : un agent peut être client dans une interaction et serveur dans une autre. |
| **Agent Card**                 | Fiche d'identité JSON publique de l'agent : nom, description, version, compétences, points d'accès, modes d'authentification.              |
| **Skill (compétence)**         | Une capacité déclarée dans l'Agent Card (ex. `book_flight`), avec description, tags, exemples et formats acceptés.                         |
| **Task (tâche)**               | Unité de travail avec un identifiant et un état, générée par le serveur. Elle peut durer quelques millisecondes ou plusieurs jours.        |
| **Message**                    | Un tour de conversation, porté par un rôle (`user` ou `agent`).                                                                            |
| **Part**                       | Un morceau de contenu dans un message : texte, données JSON structurées, URL vers un fichier, ou octets bruts.                             |
| **Artifact**                   | Le livrable produit par la tâche (un rapport, un JSON d'itinéraire, un PDF…), lui aussi composé de Parts.                                  |
| **Context**                    | Identifiant qui permet de regrouper plusieurs tâches et messages d'une même conversation.                                                  |

---

## 4. Architecture en trois couches

La spécification v1.0 est découpée en trois couches, ce qui permet de changer de transport sans réécrire la logique de l'agent :

```
┌─────────────────────────────────────────────────────────┐
│ Couche 3 — Liaisons de protocole (bindings)             │
│   JSON-RPC 2.0 sur HTTPS  │  gRPC  │  HTTP+JSON (REST)  │
├─────────────────────────────────────────────────────────┤
│ Couche 2 — Opérations abstraites                        │
│   SendMessage, SendStreamingMessage, GetTask,           │
│   ListTasks, CancelTask, SubscribeToTask,               │
│   gestion des notifications push, récupération de carte │
├─────────────────────────────────────────────────────────┤
│ Couche 1 — Modèle de données canonique                  │
│   AgentCard, AgentSkill, Task, Message, Part,           │
│   Artifact, Extension (Protocol Buffers + JSON Schema)  │
└─────────────────────────────────────────────────────────┘
```

Chaque agent annonce dans son Agent Card la ou les liaisons qu'il supporte (champ `supportedInterfaces`). JSON-RPC est la plus répandue pour les agents exposés publiquement ; gRPC est intéressant pour les échanges internes à faible latence.

---

## 5. Mode opératoire pas à pas

Prenons un scénario concret : un **agent RH** doit organiser un entretien et délègue la réservation à un **agent de voyage** géré par une autre équipe.

![Séquence d'une délégation de tâche A2A : découverte, authentification, envoi, suivi, échange multi-tours et résultat]({{ '/assets/images/sequence-a2a.svg' | relative_url }})

### Étape 1 — Découverte via l'Agent Card

L'agent distant publie sa carte à une URL standard : `https://<domaine>/.well-known/agent-card.json` (convention RFC 8615). Exemple simplifié conforme v1.0 :

```json
{
  "name": "Agent de réservation de voyages",
  "description": "Réserve des vols et hôtels pour les collaborateurs.",
  "version": "1.2.0",
  "provider": {
    "organization": "MonEntreprise",
    "url": "https://monentreprise.example"
  },
  "supportedInterfaces": [
    {
      "url": "https://voyage.monentreprise.example/a2a",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    }
  ],
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "extendedAgentCard": false
  },
  "defaultInputModes": ["text/plain", "application/json"],
  "defaultOutputModes": ["application/json"],
  "skills": [
    {
      "id": "book_flight",
      "name": "Réserver un vol",
      "description": "Réserve un vol à partir d'une origine, d'une destination et de dates.",
      "tags": ["voyage", "réservation"],
      "examples": [
        "Réserve un Paris–Lyon le 12 novembre pour le matricule 4432"
      ]
    }
  ],
  "securitySchemes": {
    "corp_oauth": {
      "type": "oauth2",
      "flows": {
        "clientCredentials": {
          "tokenUrl": "https://auth.monentreprise.example/token",
          "scopes": { "flights:write": "Réserver des vols" }
        }
      }
    }
  },
  "security": [{ "corp_oauth": ["flights:write"] }]
}
```

Le client lit la carte, vérifie que le skill `book_flight` existe, choisit une interface et identifie le mode d'authentification attendu.

La découverte peut aussi passer par un **registre** interne (catalogue d'agents validés) ou par une **configuration directe** (URL codée en dur) pour des systèmes fortement couplés.

### Étape 2 — Authentification

Les identifiants ne voyagent **jamais** dans le corps du message A2A : ils sont transmis dans les en-têtes HTTP (`Authorization: Bearer …`). Ici, le client effectue un flux OAuth2 _client credentials_ auprès du `tokenUrl` déclaré.

### Étape 3 — Envoi de la tâche (`SendMessage`)

```http
POST /a2a HTTP/1.1
Host: voyage.monentreprise.example
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json
Accept: application/json, text/event-stream
```

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "SendMessage",
  "params": {
    "message": {
      "role": "ROLE_USER",
      "parts": [
        {
          "text": "Réserve un vol direct Paris–Lyon le 12 novembre pour le matricule 4432."
        }
      ]
    },
    "configuration": { "returnImmediately": true }
  }
}
```

Points à noter :

- Le client **ne fournit pas** d'identifiant de tâche : c'est le serveur qui le génère et le renvoie.
- Pour une réponse triviale, le serveur peut renvoyer directement un `Message` au lieu de créer une `Task`.
- `returnImmediately: true` demande au serveur de rendre la main tout de suite avec la tâche créée, pour que le client puisse suivre l'avancement en parallèle (en v1.0, le comportement par défaut est bloquant).

### Étape 4 — Suivi de l'avancement

Trois mécanismes, selon la durée et la topologie :

| Mécanisme              | Quand l'utiliser                                                | Opérations                                                                                                     |
| ---------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Polling**            | Tâches courtes, client simple                                   | `GetTask` à intervalle régulier                                                                                |
| **Streaming SSE**      | Le client reste connecté et veut des mises à jour en temps réel | `SendStreamingMessage` (envoi + flux) ou `SubscribeToTask` (s'abonner à une tâche existante)                   |
| **Notifications push** | Tâches très longues, client déconnecté (mobile, serverless)     | Le client enregistre un webhook ; le serveur y POSTe les mises à jour, authentifiées (JWT signé, HMAC ou mTLS) |

Exemple de flux SSE reçu :

```text
data: {"task": {"id": "task-9876", "status": {"state": "TASK_STATE_SUBMITTED"}}}

data: {"statusUpdate": {"taskId": "task-9876", "status": {"state": "TASK_STATE_WORKING"}}}

data: {"artifactUpdate": {"taskId": "task-9876", "artifact": {"artifactId": "a1", "name": "itineraire", "parts": [ ... ]}}}

data: {"statusUpdate": {"taskId": "task-9876", "status": {"state": "TASK_STATE_COMPLETED"}}}
```

Le flux se ferme automatiquement quand la tâche atteint un état terminal.

### Étape 5 — Échange multi-tours (si nécessaire)

Si l'agent distant a besoin d'une précision (« classe éco ou affaires ? »), la tâche passe en `INPUT_REQUIRED`. Le client répond par un nouveau `SendMessage` portant le même identifiant de tâche, et le travail reprend.

### Étape 6 — Récupération du résultat

Les **artifacts** contiennent le livrable final. Un client qui a raté le flux peut toujours appeler `GetTask` pour obtenir la tâche complète, avec son historique et ses artifacts.

### Bonus : un client minimal en Python

```python
import requests

BASE = "https://voyage.monentreprise.example"

# 1. Découverte
card = requests.get(f"{BASE}/.well-known/agent-card.json", timeout=10).json()
if "book_flight" not in {s["id"] for s in card.get("skills", [])}:
    raise RuntimeError("L'agent ne propose pas la compétence book_flight")

endpoint = card["supportedInterfaces"][0]["url"]
headers = {"Authorization": "Bearer <jeton>", "Content-Type": "application/json"}

# 2. Envoi de la tâche
payload = {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "SendMessage",
    "params": {
        "message": {
            "role": "ROLE_USER",
            "parts": [{"text": "Réserve un Paris–Lyon le 12/11 pour le matricule 4432."}],
        },
        "configuration": {"returnImmediately": True},
    },
}
task = requests.post(endpoint, json=payload, headers=headers, timeout=30).json()["result"]

# 3. Lecture de l'état (polling simple)
status = requests.post(
    endpoint,
    json={"jsonrpc": "2.0", "id": 2, "method": "GetTask",
          "params": {"taskId": task["task"]["id"]}},
    headers=headers, timeout=30,
).json()
print(status)
```

> En production, préférez les SDK officiels (Python, JavaScript, Java, Go, .NET) publiés par le projet A2A plutôt que des appels HTTP à la main. La forme exacte de `result` dépend de la réponse du serveur (`task` ou `message`).

---

## 6. Cycle de vie d'une tâche

![Cycle de vie d'une tâche A2A : états en cours, interrompus et terminaux]({{ '/assets/images/cycle-de-vie-tache.svg' | relative_url }})

| État (v1.0)                 | Catégorie   | Signification                                           |
| --------------------------- | ----------- | ------------------------------------------------------- |
| `TASK_STATE_SUBMITTED`      | en cours    | Acceptée, pas encore démarrée                           |
| `TASK_STATE_WORKING`        | en cours    | Traitement actif                                        |
| `TASK_STATE_INPUT_REQUIRED` | interrompue | En attente d'une information du client                  |
| `TASK_STATE_AUTH_REQUIRED`  | interrompue | Le client doit fournir des identifiants supplémentaires |
| `TASK_STATE_COMPLETED`      | terminale   | Terminée avec succès                                    |
| `TASK_STATE_FAILED`         | terminale   | Erreur irrécupérable                                    |
| `TASK_STATE_CANCELED`       | terminale   | Annulée (souvent via `CancelTask`)                      |
| `TASK_STATE_REJECTED`       | terminale   | Refusée par l'agent (politique, quota, capacité)        |

---

## 7. Sécurité

A2A s'appuie sur des standards du web plutôt que d'inventer les siens :

- **Transport chiffré** : HTTPS (TLS 1.2 minimum) obligatoire en production.
- **Schémas d'authentification** alignés sur OpenAPI : OAuth 2.0, OpenID Connect, clé d'API, mTLS. Déclarés dans `securitySchemes` et exigés via `security`.
- **Agent Cards signées** : une signature JWS (RFC 7515) calculée sur une forme canonique de la carte (RFC 8785) permet de détecter toute falsification — une carte modifiée pourrait sinon rediriger les appels ou injecter du contenu malveillant dans le prompt de l'agent client.
- **Agent Card étendue** : un agent peut exposer une carte publique minimale et une carte plus détaillée, accessible seulement après authentification.
- **Traçabilité** : la spécification recommande OpenTelemetry et la propagation des en-têtes W3C Trace Context (`traceparent`) pour suivre une requête à travers toute la chaîne d'agents.

Pour un niveau d'assurance élevé : combiner mTLS et OAuth2, ou utiliser des jetons liés à l'émetteur (DPoP, RFC 9449), afin qu'un jeton volé soit inutilisable depuis une autre machine.

---

## 8. A2A vs MCP

### Rappel : qu'est-ce que MCP ?

**MCP (Model Context Protocol)**, lancé par Anthropic fin 2024 et désormais largement adopté, standardise la connexion entre une application IA (le _client/host_) et des **serveurs** qui exposent des **outils** (tools), des **ressources** (données) et des **prompts**. Il fonctionne sur JSON-RPC 2.0, en local via `stdio` ou à distance via HTTP. Ses spécifications sont nommées par date ; la révision **2026-07-28** a notamment rendu le protocole **sans état** (suppression des sessions et de la poignée de main d'initialisation), ce qui facilite le déploiement horizontal des serveurs.

### Tableau comparatif

| Critère                       | **A2A**                                                         | **MCP**                                                  |
| ----------------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| **Objet**                     | Faire collaborer des agents autonomes entre eux                 | Donner à un agent l'accès à des outils et des données    |
| **Analogie**                  | Un collègue à qui l'on délègue un dossier                       | Une boîte à outils que l'on manipule soi-même            |
| **Nature de l'interlocuteur** | Opaque, autonome, raisonne par lui-même                         | Fonction déterministe ou source de données exposée       |
| **Unité d'échange**           | Tâche (`Task`) avec état et cycle de vie                        | Appel d'outil / lecture de ressource                     |
| **Durée typique**             | Secondes à jours, multi-tours                                   | Généralement courte, requête/réponse                     |
| **Interaction**               | Négociation possible (`INPUT_REQUIRED`, `AUTH_REQUIRED`)        | Le modèle appelle, l'outil répond                        |
| **Découverte**                | Agent Card publique (`/.well-known/agent-card.json`), registres | Listing des outils/ressources auprès du serveur connecté |
| **Transports**                | JSON-RPC 2.0, gRPC, HTTP+JSON                                   | JSON-RPC 2.0 sur stdio ou HTTP                           |
| **Asynchrone / long**         | Natif : SSE, notifications push par webhook                     | Possible mais moins central                              |
| **Contenus**                  | Multimodal via Parts (texte, JSON, fichiers, octets)            | Résultats structurés, texte, images, ressources          |
| **Topologie**                 | Pair à pair, rôles client/serveur interchangeables              | Hôte → client → serveur                                  |
| **Origine / gouvernance**     | Google → Linux Foundation                                       | Anthropic → gouvernance ouverte de la communauté MCP     |
| **Maturité (oct. 2026)**      | v1.0 stable (mars 2026)                                         | Révision 2026-07-28 stable                               |

### La bonne question à se poser

> _« Est-ce que je veux **exécuter une action précise** ou **confier un objectif** à quelqu'un qui saura s'organiser ? »_

- **Action précise et déterministe** (requête SQL, lecture d'un fichier, appel d'API météo) → **MCP**.
- **Objectif à atteindre** par un système qui raisonne, peut poser des questions, et dont on ne veut pas connaître l'implémentation → **A2A**.

Il existe une zone grise : on peut exposer un agent complet comme un « outil » MCP. Cela fonctionne pour des cas simples, mais on perd ce qui fait la force d'A2A : le suivi d'état, les échanges multi-tours, les tâches longues et la découverte inter-organisations.

---

## 9. Utiliser A2A et MCP ensemble

Les deux protocoles sont **complémentaires** et forment ensemble la pile de communication des systèmes multi-agents :

![Un orchestrateur délègue via A2A à deux agents, qui utilisent chacun leurs outils via MCP]({{ '/assets/images/a2a-mcp-ensemble.svg' | relative_url }})

**Scénario :**

1. L'utilisateur demande à l'orchestrateur : « Organise l'entretien de Marie à Lyon jeudi. »
2. L'orchestrateur délègue via **A2A** à l'agent RH (« planifie l'entretien ») et à l'agent Voyage (« réserve le déplacement »).
3. Chacun de ces agents utilise **MCP** en interne pour interroger ses propres outils : agenda, SIRH, API de réservation.
4. Les agents renvoient leurs **artifacts** (créneau confirmé, billet) à l'orchestrateur via A2A.

Règle pratique : **A2A entre agents, MCP entre un agent et ses outils.**

---

## 10. Limites actuelles et bonnes pratiques

**Limites à connaître :**

- **Schéma des données par skill** : un skill déclare accepter `application/json`, mais la spécification ne standardise pas le schéma exact attendu. Solution : publier un JSON Schema dans la description du skill ou via une _Extension_ déclarée dans l'Agent Card.
- **Réduction des droits en cascade** : A2A dit comment transmettre des identifiants, pas comment les restreindre lorsqu'un orchestrateur très privilégié délègue à un agent moins privilégié. Solution : OAuth 2.0 Token Exchange (RFC 8693) pour émettre des jetons courts et ciblés.
- **Registre non standardisé** : la découverte via un catalogue central reste à construire soi-même.
- **Migration 0.x → 1.0** : noms de méthodes (`message/send` → `SendMessage`), noms d'états préfixés, disparition des champs `kind` et `final`.

**Bonnes pratiques :**

- Considérer l'Agent Card comme un **contrat** : tout changement de skill, d'URL ou de capacité doit incrémenter la version.
- **Signer** les Agent Cards et vérifier les signatures côté client.
- Servir la carte avec `Content-Type: application/json`.
- Placer une **passerelle API** devant les agents pour centraliser authentification, quotas, journalisation et gestion des connexions longues.
- Instrumenter avec **OpenTelemetry** dès le départ : déboguer une chaîne de cinq agents sans traces distribuées est très pénible.
- Ne jamais faire confiance aveuglément au contenu reçu d'un autre agent : c'est une donnée, pas une instruction (risque d'injection de prompt).

---

## 11. Conclusion

A2A fait pour les agents ce que HTTP a fait pour les documents ou SMTP pour les e-mails : un **langage commun et neutre** qui permet à des systèmes hétérogènes de coopérer sans intégration sur mesure. Avec sa v1.0 stable, sa gouvernance par la Linux Foundation et son adoption par les grandes plateformes, il est désormais utilisable en production.

MCP et A2A ne sont pas concurrents : **MCP donne des mains à un agent, A2A lui donne des collègues.** Une architecture multi-agents moderne utilise les deux.

---

## Pour aller plus loin

- [Spécification officielle A2A](https://a2a-protocol.org/latest/specification/)
- [A2A et MCP — relation entre les deux protocoles (site officiel A2A)](https://a2a-protocol.org/latest/topics/a2a-and-mcp/)
- [Spécification MCP](https://modelcontextprotocol.io/specification/latest)
- [Guide technique A2A v1.0 — Tyk](https://tyk.io/learning-center/a2a-protocol-architecture-and-technical-specification/)
- [MCP vs A2A — Metorial](https://metorial.com/for-ai-crawlers/mcp-vs-a2a)
