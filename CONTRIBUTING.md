# Guide de contribution — FitConnect

Ce document définit les conventions à respecter pour contribuer au projet FitConnect. Il s'applique aux deux dépôts : backend (monorepo microservices) et frontend (React SPA).

---

## Sommaire

1. [Prérequis](#1-prérequis)
2. [Branches](#2-branches)
3. [Commits](#3-commits)
4. [Ajouter un nouveau service](#4-ajouter-un-nouveau-service)
5. [Modifier un fichier proto existant](#5-modifier-un-fichier-proto-existant)
6. [Process de review](#6-process-de-review)
7. [Checklist avant PR](#7-checklist-avant-pr)

---

## 1. Prérequis

- Node.js 20+
- Docker Desktop installé et démarré
- Avoir copié tous les `.env.example` → `.env` dans chaque service
- Avoir lancé `npm run local:up` au moins une fois pour vérifier que l'environnement démarre

```bash
# Vérification rapide de l'environnement
npm run local:up
npm run local:status
npm run local:smoke
```

---

## 2. Branches

### Convention de nommage

```
<type>/<service>/<description-courte>
```

| Type | Usage |
|---|---|
| `feat` | Nouvelle fonctionnalité |
| `fix` | Correction de bug |
| `refactor` | Refactoring sans changement de comportement |
| `docs` | Documentation uniquement |
| `chore` | Tâches techniques (config, deps, docker) |
| `test` | Ajout ou modification de tests |

### Exemples

```bash
feat/planning-service/add-workout-validation
fix/chat-service/websocket-reconnect-strategy
docs/api-gateway/update-graphql-schema
chore/docker/add-healthcheck-planning
refactor/frontend/extract-websocket-hook
```

### Branches protégées

- `main` — production, merge par PR uniquement
- `develop` — branche d'intégration, base de toutes les feature branches

---

## 3. Commits

### Format — Conventional Commits

```
<type>(<scope>): <description courte>

[corps optionnel]

[footer optionnel — références issues]
```

### Exemples

```
feat(planning-service): add validateWorkoutSession gRPC endpoint

Implements the ValidateWorkout RPC defined in event.proto.
Publishes WORKOUT_COMPLETED event to Redis on success.

Closes #42
```

```
fix(chat-service): set Redis reconnectStrategy to exponential backoff

reconnectStrategy: false caused silent event loss after Redis restart.
Now retries with Math.min(retries * 100, 3000) delay.
```

```
feat(frontend): add WebSocket hook for real-time events

Centralizes WS connection lifecycle in useWebSocket.js.
Handles new_message, notification, leaderboard_update events.
```

### Règles

- Description en anglais, impératif présent ("add" pas "added")
- Maximum 72 caractères pour la première ligne
- Le scope correspond au nom du service ou du dossier concerné
- Un commit = une intention claire

---

## 4. Ajouter un nouveau service

Suivre ces étapes dans l'ordre :

### Étape 1 — Créer le fichier `.proto`

```
<nouveau-service>/proto/<nom>.proto
```

Définir les RPCs et les messages. C'est le contrat — tout le reste en découle.

```protobuf
syntax = "proto3";
package newservice;

service NewService {
  rpc DoSomething (DoSomethingRequest) returns (DoSomethingResponse);
}

message DoSomethingRequest {
  string user_id = 1;
}

message DoSomethingResponse {
  bool success = 1;
}
```

### Étape 2 — Copier le proto dans la gateway

```bash
cp <nouveau-service>/proto/<nom>.proto api-gateway/proto/<nom>.proto
```

> ⚠️ Ces deux fichiers doivent toujours être identiques. Si l'un est modifié, l'autre doit l'être aussi immédiatement.

### Étape 3 — Implémenter le serveur gRPC dans le service

```
<nouveau-service>/src/grpc/<NomService>GrpcServer.ts
```

### Étape 4 — Créer le client gRPC dans la gateway

```
api-gateway/src/clients/<nomService>Client.ts
```

### Étape 5 — Ajouter les resolvers GraphQL dans la gateway

```
api-gateway/src/graphql/resolvers/<nomService>Resolvers.ts
```

Mettre à jour le schéma GraphQL pour exposer les nouvelles queries/mutations.

### Étape 6 — Ajouter le service dans Docker Compose

Ajouter le service, sa base PostgreSQL dédiée et ses variables d'environnement dans `docker-compose.yml`.

### Étape 7 — Documenter

Créer `<nouveau-service>/README.md` en suivant le template des autres services.

---

## 5. Modifier un fichier proto existant

Les fichiers `.proto` sont des **contrats d'interface**. Une modification mal gérée casse silencieusement la communication entre la gateway et le service concerné.

### Règles obligatoires

1. **Ne jamais modifier le numéro de champ d'un message existant** — c'est ce qui identifie les données en binaire
2. **Ne jamais supprimer un champ** — préférer le déprécier avec `[deprecated = true]`
3. **Toujours mettre à jour les deux copies en même temps** : `<service>/proto/<nom>.proto` et `api-gateway/proto/<nom>.proto`
4. **Mettre à jour le stub gRPC** dans `api-gateway/src/clients/` après chaque modification proto

### Cas de l'auth-services

`auth-services` possède deux copies du proto :
- `auth-services/proto/auth.proto`
- `auth-services/src/grpc/proto/auth.proto`

Toute modification doit être répercutée sur les **trois fichiers** simultanément.

---

## 6. Process de review

### Ouvrir une Pull Request

- Base : `develop`
- Titre au format Conventional Commits : `feat(planning-service): add event creation`
- Description : contexte, ce qui a changé, comment tester
- Lier les issues concernées

### Checklist reviewer

- [ ] Le proto est-il modifié ? Si oui, les deux copies sont-elles synchronisées ?
- [ ] Les variables d'environnement nouvelles sont-elles dans `.env.example` ?
- [ ] Le `docker-compose.yml` est-il mis à jour si un nouveau service est ajouté ?
- [ ] Les smoke tests passent-ils (`npm run local:smoke`) ?
- [ ] La documentation du service concerné est-elle à jour ?

---

## 7. Checklist avant PR

```
[ ] npm run local:up     → tous les services démarrent
[ ] npm run local:status → tous les health checks passent
[ ] npm run local:smoke  → tous les tests passent
[ ] .env.example mis à jour si nouvelles variables
[ ] README du service concerné mis à jour
[ ] Deux copies du proto synchronisées si proto modifié
[ ] Pas de console.log oubliés
[ ] TypeScript compile sans erreur (npm run build dans le service)
```
