# API Gateway — FitConnect

> Point d'entrée unique de toute l'application FitConnect.  
> Stack : Apollo Server 4 · GraphQL · gRPC · JWT middleware  
> Port : **4100**

---

## Sommaire

1. [Rôle et responsabilités](#1-rôle-et-responsabilités)
2. [Structure du projet](#2-structure-du-projet)
3. [Schéma GraphQL](#3-schéma-graphql)
4. [Clients gRPC](#4-clients-grpc)
5. [Authentification JWT](#5-authentification-jwt)
6. [Ajouter un resolver](#6-ajouter-un-resolver)
7. [Ajouter un client gRPC](#7-ajouter-un-client-grpc)
8. [Variables d'environnement](#8-variables-denvironnement)
9. [Démarrage](#9-démarrage)

---

## 1. Rôle et responsabilités

La gateway est le **seul service exposé publiquement**. Le frontend ne connaît que son adresse.

Elle a trois responsabilités précises :

- **Exposer l'API GraphQL** — elle agrège les données de tous les services en une interface unifiée
- **Vérifier le JWT** — sur chaque requête, avant tout appel gRPC
- **Orchestrer les appels gRPC** — elle traduit chaque query/mutation GraphQL en un ou plusieurs appels vers les services concernés

Elle ne contient **aucune logique métier**. Toute validation, persistance et règle de domaine se trouve dans les services.

```
Frontend
   │ GraphQL HTTP POST :4100
   ▼
API Gateway
   │ Vérifie JWT (Redis cache → Auth Service)
   │
   ├─► gRPC :5106 → Auth Service
   ├─► gRPC :5101 → Community Service
   ├─► gRPC :5103 → Planning Service
   ├─► gRPC :5105 → Challenge & Ranking Service
   └─► gRPC :5104 → Chat & Notification Service
```

---

## 2. Structure du projet

```
api-gateway/
├── proto/                        # Copies des .proto de chaque service
│   ├── auth.proto
│   ├── community.proto
│   ├── event.proto               # Planning Service (attention : pas planning.proto)
│   ├── ranking.proto
│   └── chat.proto
├── src/
│   ├── graphql/
│   │   ├── schema/               # Définitions des types GraphQL
│   │   └── resolvers/            # Resolvers par domaine
│   │       ├── authResolvers.ts
│   │       ├── communityResolvers.ts
│   │       ├── planningResolvers.ts
│   │       ├── rankingResolvers.ts
│   │       └── chatResolvers.ts
│   ├── clients/                  # Stubs gRPC vers chaque service
│   │   ├── authClient.ts
│   │   ├── communityClient.ts
│   │   ├── planningClient.ts
│   │   ├── rankingClient.ts
│   │   └── chatClient.ts
│   └── middleware/
│       └── authMiddleware.ts     # Vérification JWT sur chaque requête
├── .env.example
└── package.json
```

> ⚠️ Les fichiers dans `proto/` sont des **copies** des protos de chaque service. Si un proto de service est modifié, sa copie ici doit être mise à jour immédiatement.

---

## 3. Schéma GraphQL

### Auth

```graphql
type Mutation {
  signUp(email: String!, password: String!, name: String!): AuthPayload!
  signIn(email: String!, password: String!): AuthPayload!
}

type Query {
  me: User!
}

type AuthPayload {
  token: String!
  user: User!
}

type User {
  id: ID!
  name: String!
  email: String!
}
```

### Community

```graphql
type Mutation {
  createGroup(name: String!): Group!
  inviteMember(groupId: ID!, userId: ID!): Member!
}

type Query {
  getGroup(groupId: ID!): Group!
  getMembers(groupId: ID!): [Member!]!
}

type Group {
  id: ID!
  name: String!
  members: [Member!]!
}

type Member {
  id: ID!
  name: String!
  role: Role!
}

enum Role {
  ADMIN
  MEMBER
}
```

### Planning & Événements

```graphql
type Mutation {
  createWorkoutSession(date: String!, description: String!): WorkoutSession!
  validateWorkoutSession(sessionId: ID!): WorkoutSession!
  createEvent(title: String!, date: String!, location: String!, description: String!): Event!
  joinEvent(eventId: ID!): EventParticipant!
}

type Query {
  getWorkoutSessions(groupId: ID!): [WorkoutSession!]!
  getEvents(groupId: ID!): [Event!]!
}

type WorkoutSession {
  id: ID!
  date: String!
  description: String!
  status: SessionStatus!
  userId: ID!
}

type Event {
  id: ID!
  title: String!
  date: String!
  location: String!
  participants: [Member!]!
  isPast: Boolean!
}

enum SessionStatus {
  PENDING
  COMPLETED
}
```

### Challenge & Ranking

```graphql
type Mutation {
  createChallenge(title: String!, description: String!, endDate: String!): Challenge!
  joinChallenge(challengeId: ID!): ChallengeParticipation!
}

type Query {
  getLeaderboard(groupId: ID!): [RankingEntry!]!
  getUserScore(userId: ID!): Score!
  getChallenges(groupId: ID!): [Challenge!]!
}

type RankingEntry {
  rank: Int!
  user: User!
  points: Int!
  title: String
}

type Score {
  points: Int!
  title: String
}
```

### Chat & Notifications

```graphql
type Mutation {
  sendMessage(groupId: ID!, content: String!): Message!
}

type Query {
  getMessages(groupId: ID!): [Message!]!
  getNotifications(userId: ID!): [Notification!]!
}

type Message {
  id: ID!
  content: String!
  author: User!
  createdAt: String!
}

type Notification {
  id: ID!
  type: String!
  content: String!
  read: Boolean!
  createdAt: String!
}
```

---

## 4. Clients gRPC

Chaque client gRPC dans `src/clients/` est initialisé avec l'adresse du service correspondant et le fichier `.proto` associé.

```typescript
// Exemple — planningClient.ts
import * as grpc from '@grpc/grpc-js'
import * as protoLoader from '@grpc/proto-loader'
import path from 'path'

const packageDef = protoLoader.loadSync(
  path.resolve(__dirname, '../../proto/event.proto')
)
const proto = grpc.loadPackageDefinition(packageDef) as any

const client = new proto.planning.PlanningService(
  process.env.PLANNING_GRPC_URL,
  grpc.credentials.createInsecure()
)

export default client
```

### Adresses gRPC des services

| Service | Variable d'env | Adresse par défaut |
|---|---|---|
| Auth Service | `AUTH_GRPC_URL` | `localhost:5106` |
| Community Service | `COMMUNITY_GRPC_URL` | `localhost:5101` |
| Planning Service | `PLANNING_GRPC_URL` | `localhost:5103` |
| Challenge Service | `CHALLENGE_GRPC_URL` | `localhost:5105` |
| Chat Service | `CHAT_GRPC_URL` | `localhost:5104` |

---

## 5. Authentification JWT

Le middleware `src/middleware/authMiddleware.ts` est appliqué à chaque requête GraphQL avant l'exécution des resolvers.

### Flux de vérification

```
Requête GraphQL reçue
        │
        ▼
Extraction du token depuis Authorization: Bearer <token>
        │
        ▼
Lookup dans Redis Cache (clé: hash du token)
        │
    ┌───┴───┐
  HIT       MISS
    │         │
    ▼         ▼
Token OK   Appel gRPC → Auth Service → ValidateToken
    │         │
    │         ▼
    │    Résultat stocké dans Redis (TTL: 1h)
    │         │
    └────┬────┘
         ▼
  userId injecté dans le contexte GraphQL
  → disponible dans tous les resolvers via context.userId
```

### Routes publiques (sans JWT)

- `signUp`
- `signIn`

Toutes les autres queries et mutations requièrent un JWT valide.

---

## 6. Ajouter un resolver

### Étape 1 — Ajouter le type dans le schéma

Dans `src/graphql/schema/`, ajouter la query ou mutation au type correspondant.

### Étape 2 — Implémenter le resolver

```typescript
// src/graphql/resolvers/planningResolvers.ts
import planningClient from '../../clients/planningClient'

export const planningResolvers = {
  Mutation: {
    createWorkoutSession: async (_: any, args: any, context: any) => {
      const { userId } = context // injecté par authMiddleware

      return new Promise((resolve, reject) => {
        planningClient.CreateWorkoutSession(
          { userId, ...args },
          (err: any, response: any) => {
            if (err) reject(err)
            else resolve(response)
          }
        )
      })
    }
  }
}
```

### Étape 3 — Merger les resolvers

Dans `src/graphql/resolvers/index.ts`, s'assurer que le nouveau resolver est bien mergé avec les autres.

---

## 7. Ajouter un client gRPC

1. Copier le `.proto` du nouveau service dans `api-gateway/proto/`
2. Créer `src/clients/<nomService>Client.ts` en suivant le pattern existant
3. Ajouter la variable d'environnement `<NOM>_GRPC_URL` dans `.env.example`
4. Créer les resolvers GraphQL correspondants dans `src/graphql/resolvers/`

---

## 8. Variables d'environnement

```env
# Serveur
PORT=4100

# JWT
JWT_SECRET=your_jwt_secret

# Redis (cache JWT)
REDIS_URL=redis://localhost:6379

# Adresses gRPC des services
AUTH_GRPC_URL=localhost:5106
COMMUNITY_GRPC_URL=localhost:5101
PLANNING_GRPC_URL=localhost:5103
CHALLENGE_GRPC_URL=localhost:5105
CHAT_GRPC_URL=localhost:5104
```

---

## 9. Démarrage

```bash
# Développement (hot-reload)
npm run gateway:dev

# Production
npm run gateway:build
npm run gateway:start

# Depuis Docker (recommandé)
npm run local:up
```

### Tester le GraphQL

Une fois démarré, l'interface Apollo Sandbox est accessible sur `http://localhost:4100`.

Exemple de requête de test :

```graphql
query {
  __typename
}
```

Doit retourner `{ "data": { "__typename": "Query" } }` si la gateway est opérationnelle.
