# Hanim MOUSSA YARI


# FitConnect — Architecture du Projet

> Plateforme communautaire sportive · M2 Web Services  
> Stack : React · Apollo Server 4 · gRPC · Redis · PostgreSQL · WebSocket · Docker

---

## Sommaire

1. [Vision du projet](#1-vision-du-projet)
2. [Stack technique](#2-stack-technique)
3. [Architecture générale](#3-architecture-générale)
4. [Découpage microservices](#4-découpage-microservices)
5. [Communication inter-services](#5-communication-inter-services)
6. [Temps réel — WebSocket](#6-temps-réel--websocket)
7. [Redis — Cache & Pub/Sub](#7-redis--cache--pubsub)
8. [Base de données](#8-base-de-données)
9. [Authentification & Sécurité](#9-authentification--sécurité)
10. [Dockerisation](#10-dockerisation)
11. [Démarrage local](#11-démarrage-local)
12. [Variables d'environnement](#12-variables-denvironnement)

---

## 1. Vision du projet

FitConnect résout un problème concret : les groupes d'amis sportifs manquent d'un outil centralisé pour coordonner leurs entraînements. Les plannings sont éparpillés, l'organisation de sessions collectives est laborieuse, et il n'existe pas de mécanique de motivation partagée pour maintenir l'engagement dans la durée.

**Fonctionnalités principales (V1) :**

- Gestion de groupe avec rôles Admin / Membre
- Planning personnel partagé avec validation de séances
- Événements collectifs avec inscription
- Challenges mensuels avec suivi de participation
- Classement basé sur l'assiduité avec titres gamifiés (Machine, Discipline Master, Iron Mind, MVP du Mois)
- Chat de groupe en temps réel avec notifications automatiques

---

## 2. Stack technique

| Couche | Technologie | Rôle |
|---|---|---|
| Frontend | React 18 + Vite | SPA, 9 pages, JWT localStorage |
| API publique | GraphQL (Apollo Server 4) | Point d'entrée unique, agrégation |
| Communication interne | gRPC (`@grpc/grpc-js`) | Appels typés inter-services |
| Bus d'événements | Redis Pub/Sub | Découplage asynchrone |
| Cache | Redis | Sessions JWT, données chaudes |
| Temps réel | WebSocket (`ws`) | Chat, notifications, leaderboard live |
| Base de données | PostgreSQL (TypeORM) | Persistance relationnelle |
| Orchestration | Docker Compose | Environnement reproductible |
| Langage | TypeScript 5.x strict | Tout le backend |

---

## 3. Architecture générale

![Texte alternatif](./PostgreSQL Microservices-2026-04-28-103144.png)


> **Règle fondamentale** : le frontend ne connaît que l'API Gateway. Le WebSocket vers le Chat Service est la seule exception — intentionnelle, car les connexions persistantes n'ont pas leur place dans une gateway de requêtes ponctuelles.

---

## 4. Découpage microservices

### 4.1 API Gateway
**Port HTTP :** 4100

Point d'entrée unique de toute l'application. Il ne contient aucune logique métier — son rôle est d'exposer le schéma GraphQL, de vérifier le JWT sur chaque requête, et de déléguer chaque opération au service concerné via gRPC.

**Fichiers clés :**
```
api-gateway/
├── src/graphql/        # Schéma + resolvers GraphQL
├── src/clients/        # Stubs gRPC vers chaque service
├── src/middleware/     # Vérification JWT
└── proto/              # Copies des .proto de chaque service
```

---

### 4.2 Auth Service
**Port HTTP :** 4102 · **Port gRPC :** 5106

Gère l'intégralité du cycle de vie des identités : inscription, connexion, génération et validation des JWT. Utilise bcrypt pour le hachage des mots de passe et Redis pour mettre en cache les sessions validées, évitant des allers-retours en base à chaque requête.

**Proto :** `auth.proto`  
**Endpoints exposés :** `SignUp`, `SignIn`, `ValidateToken`

> ⚠️ `auth-services` contient deux copies du proto (`src/grpc/proto/auth.proto` et `proto/auth.proto`). Les garder synchronisées est critique.

---

### 4.3 Community Service
**Port HTTP :** 3001 · **Port gRPC :** 5101

Seul service utilisant **NestJS** (les autres utilisent Express). Gère les groupes, les membres et leurs rôles. Utilise TypeORM avec `synchronize: true` en développement.

**Proto :** `community.proto`  
**Tables :** `group`, `user`, `membership`  
**Endpoints exposés :** `CreateGroup`, `InviteMember`, `GetGroup`, `GetMembers`

> ⚠️ `group` est un mot réservé PostgreSQL. TypeORM le quote automatiquement (`"group"`), mais à surveiller si du SQL brut est écrit.

---

### 4.4 Planning & Event Service
**Port HTTP :** 4103 · **Port gRPC :** 5103

Gère les séances d'entraînement individuelles et les événements collectifs. C'est le **seul publisher Redis** de l'architecture — chaque action significative déclenche un événement asynchrone consommé par les autres services.

**Proto :** `event.proto` (attention : pas `planning.proto`)  
**Tables :** `workout_sessions`, `events`, `event_participants`  
**Endpoints exposés :** `CreateWorkoutSession`, `ValidateWorkoutSession`, `CreateEvent`, `JoinEvent`, `GetEvents`

**Flux Redis publiés :**

| Action | Canal publié | Consommateurs |
|---|---|---|
| Séance validée | `WORKOUT_COMPLETED` | Challenge Service, Chat Service |
| Événement créé | `EVENT_CREATED` | Challenge Service, Chat Service |

---

### 4.5 Challenge & Ranking Service
**Port gRPC :** 5105

Calcule les points, gère les challenges mensuels, génère le leaderboard et attribue les titres automatiquement selon le score accumulé.

**Proto :** `ranking.proto`  
**Tables :** `challenges`, `challenge_participations`, `scores`, `titles`  
**Endpoints exposés :** `GetLeaderboard`, `GetUserScore`, `CreateChallenge`, `JoinChallenge`

**Titres attribués automatiquement :**

| Titre | Condition |
|---|---|
| 🏅 Machine | Assiduité élevée |
| 🎯 Discipline Master | Constance sur la durée |
| 🧠 Iron Mind | Challenges complétés |
| 🏆 MVP du Mois | Meilleur score mensuel |

**Souscriptions Redis :** `WORKOUT_COMPLETED`, `EVENT_CREATED`

---

### 4.6 Chat & Notification Service
**Port gRPC :** 5104

Seul service exposant un **serveur WebSocket**. Gère l'historique des messages et les notifications. Reçoit les événements Redis et les pousse en temps réel aux clients connectés via WebSocket.

**Proto :** `chat.proto`  
**Tables :** `messages`, `notifications`  
**Endpoints GraphQL :** `GetMessages`, `SendMessage`, `GetNotifications`

**Événements WebSocket émis :**

| Événement WS | Déclenché par | Effet sur le frontend |
|---|---|---|
| `new_message` | `SendMessage` mutation | Nouveau message dans le chat |
| `notification` | Redis `WORKOUT_COMPLETED` / `EVENT_CREATED` | Toast de notification |
| `leaderboard_update` | Redis `WORKOUT_COMPLETED` | Mise à jour du classement en live |

> ⚠️ `reconnectStrategy: false` dans ce service. Si Redis redémarre, le service ne tente pas de se reconnecter automatiquement et rate silencieusement tous les événements suivants. À corriger en production.

---

## 5. Communication inter-services

### gRPC — synchrone

Utilisé pour toutes les communications **API Gateway → Services**. Chaque service expose un serveur gRPC défini par un fichier `.proto` qui constitue le contrat d'interface.

```
Client → GraphQL → Gateway → gRPC stub → Service → gRPC response → Gateway → GraphQL response → Client
```

**Avantages vs REST interne :**
- Sérialisation binaire (Protocol Buffers) — 5 à 10× plus compact que JSON
- Contrats typés — impossible de casser une interface sans mettre à jour le proto
- HTTP/2 — multiplexage, compression de headers

**Fichiers proto par service :**

| Service | Proto |
|---|---|
| auth-services | `auth.proto` |
| community-service | `community.proto` |
| planning-service | `event.proto` |
| challenge-ranking-service | `ranking.proto` |
| chat-notification-service | `chat.proto` |

---

### Redis Pub/Sub — asynchrone

Utilisé pour les communications **inter-services sans couplage direct**. Le Planning Service publie un événement sans savoir combien de services y réagissent.

```
Planning Service
      │
      └─► PUBLISH WORKOUT_COMPLETED ──► Redis
                                          ├─► Challenge Service (recalcule les points)
                                          └─► Chat Service (envoie notification WS)
```

Ce pattern garantit que :
- Le Planning Service répond immédiatement à l'utilisateur sans attendre les traitements secondaires
- Ajouter un nouveau service consommateur ne nécessite aucune modification du publisher
- La panne d'un subscriber n'affecte pas les autres

---

## 6. Temps réel — WebSocket

Le Chat & Notification Service maintient des connexions WebSocket persistantes avec le frontend. C'est le **seul canal qui bypasse l'API Gateway** — intentionnellement, car une gateway de requêtes ponctuelles n'est pas conçue pour maintenir des milliers de connexions longues.

**Connexion côté frontend :**
```javascript
const ws = new WebSocket('ws://localhost:PORT')

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)

  if (data.type === 'new_message')       { /* ajouter message au chat */ }
  if (data.type === 'notification')      { /* dispatcher fitconnect:toast */ }
  if (data.type === 'leaderboard_update') { /* mettre à jour le state classement */ }
}
```

**Cycle de vie complet d'une séance validée :**
```
1. User clique "Valider"
2. Frontend → mutation GraphQL validateWorkoutSession
3. Gateway → gRPC → Planning Service
4. Planning Service → PUBLISH WORKOUT_COMPLETED sur Redis
5. Challenge Service reçoit → recalcule points + titre
6. Chat Service reçoit → push WebSocket { type: 'notification' } + { type: 'leaderboard_update' }
7. Frontend reçoit WS → toast affiché + classement mis à jour
   → sans aucun rechargement de page
```

---

## 7. Redis — Cache & Pub/Sub

Redis joue deux rôles distincts dans l'architecture.

### Cache (Auth Service)

Chaque token JWT validé est mis en cache avec un TTL. Les requêtes suivantes utilisent le cache en mémoire au lieu d'interroger PostgreSQL.

```
Requête 1 → validation JWT → résultat stocké dans Redis (TTL: 1h)
Requête 2 → lecture Redis → réponse en < 1ms
```

### Pub/Sub (bus d'événements)

Redis sert de **broker de messages** entre les services. Les deux canaux actifs :

| Canal | Publisher | Subscribers |
|---|---|---|
| `WORKOUT_COMPLETED` | planning-service | challenge-ranking-service, chat-notification-service |
| `EVENT_CREATED` | planning-service | challenge-ranking-service, chat-notification-service |

---

## 8. Base de données

Chaque microservice possède **sa propre base PostgreSQL isolée**. Aucun service n'accède directement aux tables d'un autre — les données cross-service transitent uniquement par gRPC.

| Service | Tables principales |
|---|---|
| auth-services | users, sessions |
| community-service | group, user, membership |
| planning-service | workout_sessions, events, event_participants |
| challenge-ranking-service | challenges, challenge_participations, scores, titles |
| chat-notification-service | messages, notifications |

**TypeORM (community-service) :**
```typescript
synchronize: configService.get<string>('NODE_ENV') !== 'production'
// dev  → tables créées automatiquement
// prod → migrations explicites requises
```

Pour générer les migrations avant un déploiement :
```bash
cd community-service
npx typeorm migration:generate src/migrations/InitialSchema -d <data-source-config>
```

---

## 9. Authentification & Sécurité

- **JWT** généré à l'inscription/connexion par l'Auth Service
- **Stockage** côté frontend dans `localStorage` via `authSession.js`
- **Transmission** : header `Authorization: Bearer <token>` sur chaque requête GraphQL
- **Validation** : middleware dans l'API Gateway, résultat mis en cache dans Redis
- **Expiration** : l'événement custom `fitconnect:unauthenticated` est dispatché sur toute erreur `UNAUTHENTICATED` GraphQL — `App.jsx` écoute cet événement, vide la session et redirige vers la page de login

---

## 10. Dockerisation

Un fichier `docker-compose.yml` unique orchestre l'intégralité de l'infrastructure :

```
Services applicatifs :
  frontend               (React + Vite)
  api-gateway            (:4100)
  auth-services          (:4102)
  community-service      (:3001)
  planning-service       (:4103)
  challenge-ranking-service
  chat-notification-service

Infrastructure :
  redis                  (:6379)
  postgres-auth          (:5432)
  postgres-community     (:5433)
  postgres-planning      (:5434)
  postgres-challenge     (:5435)
  postgres-chat          (:5436)
```

---

## 11. Démarrage local

### Via Docker (recommandé)
```bash
npm run local:up        # Lance tous les services
npm run local:status    # Vérifie que tous les services répondent
npm run local:smoke     # Exécute les tests d'intégration
npm run local:down      # Arrête tout
```

### En développement (hot-reload)
```bash
npm run gateway:dev
npm run auth:dev
npm run community:dev
npm run planning:dev
npm run challenge:dev
npm run chatnotif:dev
```

### Prérequis smoke tests
- Docker démarré avec un conteneur Redis nommé exactement `fitconnect-redis`
- Tous les services démarrés et healthcheck OK
- Base auth prête (le `signUp` du smoke test en dépend)

---

## 12. Variables d'environnement

Copier `.env.example` → `.env` dans chaque répertoire de service avant de démarrer.

**Pattern commun à tous les services :**
```env
PORT=<http_port>
GRPC_PORT=<grpc_port>
DATABASE_URL=postgresql://user:pass@localhost:5432/dbname
REDIS_URL=redis://localhost:6379
JWT_SECRET=<secret>

# Adresses gRPC des services voisins (gateway uniquement)
AUTH_GRPC_URL=localhost:5106
COMMUNITY_GRPC_URL=localhost:5101
PLANNING_GRPC_URL=localhost:5103
CHALLENGE_GRPC_URL=localhost:5105
CHAT_GRPC_URL=localhost:5104

# Frontend
VITE_GATEWAY_URL=http://localhost:4100
VITE_AUTH_URL=http://localhost:4102/graphql
```

---

## Points d'attention avant la production

| Sujet | Risque | Action requise |
|---|---|---|
| Proto en double dans auth-services | Divergence silencieuse | Symlink ou script de copie au build |
| Migrations TypeORM absentes | Crash au démarrage prod | `typeorm migration:generate` |
| `reconnectStrategy: false` dans chat-service | Perte silencieuse d'événements Redis | Implémenter une stratégie de retry |
| Délai fixe 3s dans smoke tests | Flakiness en CI | Remplacer par un poll avec retry |
