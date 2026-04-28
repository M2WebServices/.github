# Frontend — FitConnect

> SPA React communautaire pour la coordination sportive entre amis.  
> Stack : React 18 · Vite · CSS pur · Iconify · GraphQL · WebSocket  
> UI entièrement en français.

---

## Sommaire

1. [Structure du projet](#1-structure-du-projet)
2. [Routing — le système activePage](#2-routing--le-système-activepage)
3. [Les 9 pages](#3-les-9-pages)
4. [Couche API GraphQL](#4-couche-api-graphql)
5. [WebSocket — temps réel](#5-websocket--temps-réel)
6. [Authentification & Session JWT](#6-authentification--session-jwt)
7. [Système d'événements custom](#7-système-dévénements-custom)
8. [UI & Composants](#8-ui--composants)
9. [Variables d'environnement](#9-variables-denvironnement)
10. [Démarrage](#10-démarrage)

---

## 1. Structure du projet

```
src/
├── App.jsx                        # Racine — routing activePage + listeners globaux
├── App.css                        # Styles globaux de l'app
├── index.css                      # Reset + variables CSS
│
├── components/                    # Composants réutilisables
│   ├── Sidebar.jsx                # Navigation principale → setActivePage
│   ├── Icon.jsx                   # Wrapper Iconify
│   └── ToastViewport.jsx          # Affichage des toasts (fitconnect:toast)
│
├── pages/                         # Une page = un composant
│   ├── AuthPage.jsx
│   ├── DashboardPage.jsx
│   ├── CommunityPage.jsx
│   ├── PlanningPage.jsx
│   ├── EventsPage.jsx
│   ├── ChallengePage.jsx
│   ├── RankingPage.jsx
│   ├── ChatPage.jsx
│   └── NotificationsPage.jsx
│
└── services/                      # Couche API et utilitaires
    ├── graphqlClient.js            # Client GraphQL de base (POST + Bearer token)
    ├── authSession.js              # Gestion JWT en localStorage
    ├── appEvents.js                # Définition des événements custom window
    ├── serviceUtils.js             # requireAuthToken, toUserError
    ├── authService.js
    ├── communityService.js
    ├── planningService.js
    ├── rankingService.js
    └── chatService.js
```

---

## 2. Routing — le système activePage

Il n'y a **pas de librairie de routing** (pas de React Router). Le routing est géré par un état local dans `App.jsx` :

```jsx
const [activePage, setActivePage] = useState('dashboard')

// Rendu conditionnel selon activePage
if (activePage === 'community') return <CommunityPage />
if (activePage === 'planning')  return <PlanningPage />
// ...
```

La `Sidebar` reçoit `setActivePage` en prop et l'appelle au clic sur chaque lien de navigation.

### Limitations connues

- Pas de support du bouton retour navigateur
- Pas de deep linking (impossible de partager une URL vers une page précise)
- L'état de chaque page est réinitialisé à chaque changement de page

---

## 3. Les 9 pages

| Clé `activePage` | Composant | Service(s) appelé(s) | WebSocket |
|---|---|---|---|
| `auth` | `AuthPage` | `authService` | ❌ |
| `dashboard` | `DashboardPage` | Plusieurs (agrégation) | ❌ |
| `community` | `CommunityPage` | `communityService` | ❌ |
| `planning` | `PlanningPage` | `planningService` | ❌ |
| `events` | `EventsPage` | `planningService` | ❌ |
| `challenges` | `ChallengePage` | `rankingService` | ❌ |
| `ranking` | `RankingPage` | `rankingService` | ✅ `leaderboard_update` |
| `chat` | `ChatPage` | `chatService` | ✅ `new_message` |
| `notifications` | `NotificationsPage` | `chatService` | ✅ `notification` |

### Pattern commun à chaque page

Chaque page gère son propre état et fetche ses données au montage :

```jsx
const [data, setData]       = useState(null)
const [loading, setLoading] = useState(true)
const [error, setError]     = useState(null)

useEffect(() => {
  planningService.getSessions()
    .then(setData)
    .catch(setError)
    .finally(() => setLoading(false))
}, [])
```

Pas de store centralisé — chaque page est autonome.

---

## 4. Couche API GraphQL

### graphqlClient.js

Toutes les requêtes GraphQL passent par ce client de base :

```javascript
// src/services/graphqlClient.js
const GATEWAY_URL = import.meta.env.VITE_GATEWAY_URL || 'http://localhost:4100'

export async function graphqlRequest(query, variables = {}) {
  const token = authSession.getToken()

  const response = await fetch(GATEWAY_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {})
    },
    body: JSON.stringify({ query, variables })
  })

  const { data, errors } = await response.json()

  if (errors) {
    // Détecte une session expirée et redirige vers le login
    if (errors.some(e => e.extensions?.code === 'UNAUTHENTICATED')) {
      window.dispatchEvent(new Event('fitconnect:unauthenticated'))
    }
    throw new Error(errors[0].message)
  }

  return data
}
```

### Services par domaine

Chaque fichier service compose des queries/mutations et appelle `graphqlRequest` :

```javascript
// src/services/planningService.js
import { graphqlRequest } from './graphqlClient.js'

export const planningService = {
  getSessions: () => graphqlRequest(`
    query {
      getWorkoutSessions {
        id date description status
      }
    }
  `),

  validateSession: (sessionId) => graphqlRequest(`
    mutation ValidateSession($sessionId: ID!) {
      validateWorkoutSession(sessionId: $sessionId) {
        id status
      }
    }
  `, { sessionId })
}
```

### serviceUtils.js

Deux helpers disponibles dans tous les services :

```javascript
requireAuthToken()  // Lance une erreur si pas de token en localStorage
toUserError(err)    // Normalise les erreurs GraphQL en message lisible
```

---

## 5. WebSocket — temps réel

### Connexion

La connexion WebSocket est ouverte **une seule fois au login** et reste active pendant toute la session. Elle se connecte directement au **Chat & Notification Service** (bypass de la gateway).

```javascript
// Initialisation après signIn réussi
const ws = new WebSocket(import.meta.env.VITE_WS_URL || 'ws://localhost:<PORT_CHAT>')

ws.onopen = () => console.log('WS connecté')
ws.onclose = () => console.log('WS déconnecté')

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)

  switch (data.type) {
    case 'new_message':
      // Ajouter le message au state de ChatPage
      // Via un event custom ou un state partagé
      break

    case 'notification':
      // Déclencher un toast
      window.dispatchEvent(new CustomEvent('fitconnect:toast', {
        detail: { message: data.content }
      }))
      break

    case 'leaderboard_update':
      // Mettre à jour le classement dans RankingPage
      break
  }
}
```

### Événements reçus

| Type | Déclenché par | Action frontend |
|---|---|---|
| `new_message` | Un membre envoie un message | Ajouter le message en bas du chat, auto-scroll |
| `notification` | `WORKOUT_COMPLETED` ou `EVENT_CREATED` Redis | Afficher un toast via `fitconnect:toast` |
| `leaderboard_update` | `WORKOUT_COMPLETED` Redis | Rafraîchir le state du leaderboard |

### Fermeture propre

```javascript
// À la déconnexion de l'utilisateur
ws.close()
```

---

## 6. Authentification & Session JWT

### authSession.js

Gère le cycle de vie du token JWT dans `localStorage` :

```javascript
authSession.setToken(token)   // Stocke après signIn/signUp
authSession.getToken()        // Récupère pour les requêtes
authSession.clearToken()      // Vide à la déconnexion
authSession.isAuthenticated() // Boolean — token présent ?
```

### Redirection automatique

`App.jsx` écoute l'événement `fitconnect:unauthenticated` dispatché par `graphqlClient.js` sur toute erreur GraphQL `UNAUTHENTICATED` :

```jsx
useEffect(() => {
  const handler = () => {
    authSession.clearToken()
    setActivePage('auth')
  }
  window.addEventListener('fitconnect:unauthenticated', handler)
  return () => window.removeEventListener('fitconnect:unauthenticated', handler)
}, [])
```

---

## 7. Système d'événements custom

Deux événements custom circulent via `window` pour la communication cross-composants, définis dans `src/services/appEvents.js` :

| Événement | Dispatché par | Écouté par | Usage |
|---|---|---|---|
| `fitconnect:unauthenticated` | `graphqlClient.js` | `App.jsx` | Session expirée → redirect login |
| `fitconnect:toast` | Partout (WS, services) | `ToastViewport.jsx` | Afficher une notification UI |

```javascript
// Dispatcher un toast depuis n'importe où
window.dispatchEvent(new CustomEvent('fitconnect:toast', {
  detail: { message: 'Séance validée !', type: 'success' }
}))
```

---

## 8. UI & Composants

### Icônes — Iconify

Les icônes sont chargées depuis le CDN Iconify dans `index.html`. Le composant `Icon.jsx` est un wrapper :

```jsx
// src/components/Icon.jsx
export function Icon({ name, size = 24 }) {
  return <iconify-icon icon={name} width={size} height={size} />
}

// Usage
<Icon name="mdi:dumbbell" size={20} />
```

### CSS

Pas de framework CSS. Deux fichiers globaux :
- `index.css` — reset, variables CSS (couleurs, espacements, typographie)
- `App.css` — layout principal (sidebar + contenu)

Chaque page peut avoir son propre fichier CSS scopé.

### ToastViewport

`ToastViewport.jsx` écoute `fitconnect:toast` et affiche les notifications en overlay :

```jsx
// Les toasts disparaissent automatiquement après un délai
// Positionnés en bas à droite de l'écran
```

---

## 9. Variables d'environnement

```env
# URL de l'API Gateway (GraphQL)
VITE_GATEWAY_URL=http://localhost:4100

# URL du service Auth (GraphQL direct, pour signIn/signUp)
VITE_AUTH_URL=http://localhost:4102/graphql

# URL WebSocket du Chat & Notification Service
VITE_WS_URL=ws://localhost:<PORT_CHAT>
```

Toutes les variables Vite doivent être préfixées `VITE_` pour être accessibles dans le code via `import.meta.env.VITE_*`.

---

## 10. Démarrage

```bash
# Développement (hot-reload)
npm run dev
# → http://localhost:5173

# Build production
npm run build

# Prévisualiser le build
npm run preview
```

### Prérequis

Le backend doit être démarré avant le frontend :

```bash
# Dans le repo backend
npm run local:up
npm run local:status  # Vérifier que tous les services répondent

# Puis dans le repo frontend
npm run dev
```
