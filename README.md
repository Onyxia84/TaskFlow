# TaskFlow

Application web de gestion de tâches en mode Kanban. Développée dans le cadre du projet final DevOps — Bachelor 3.

L'application permet de créer, filtrer et prioriser des tâches réparties en trois colonnes (À faire / En cours / Terminées), avec authentification JWT, statistiques en temps réel, monitoring Kubernetes et routing Ingress nginx.

---

## Stack technique

| Couche    | Technologie               | Rôle                                        |
|-----------|---------------------------|---------------------------------------------|
| Frontend  | HTML / CSS / JS vanilla   | Interface Kanban, servie par Nginx Alpine   |
| Backend   | Node.js 20 (sans framework) | API REST — logique métier, auth JWT       |
| Cache     | Redis 7 Alpine            | Persistance des tâches et compteurs stats   |
| Reverse proxy | Nginx Alpine          | Sert le frontend, compresse gzip, cache assets |
| CI/CD     | GitHub Actions            | Tests, lint, audit, build Docker, déploiement |
| Orchestration | Kubernetes            | Staging (2 replicas) et Production (3 replicas) |

---

## Prérequis

- [Docker](https://docs.docker.com/get-docker/) 24+
- [Docker Compose](https://docs.docker.com/compose/) v2+
- (Optionnel) [kubectl](https://kubernetes.io/docs/tasks/tools/) + [minikube](https://minikube.sigs.k8s.io/) pour Kubernetes

---

## Lancer en local avec Docker Compose

### 1. Cloner le projet

```bash
git clone https://github.com/Onyxia84/TaskFlow.git
cd TaskFlow
```

### 2. Créer le fichier de configuration

```bash
cp .env.example .env
```

Éditez `.env` si besoin (voir section [Variables d'environnement](#variables-denvironnement)).

### 3. Construire et démarrer les conteneurs

```bash
docker compose up --build
```

### 4. Accéder à l'application

| Service   | URL                          |
|-----------|------------------------------|
| Frontend  | http://localhost/            |
| Backend   | http://localhost:3001/health |

Pour arrêter :

```bash
docker compose down
```

Pour arrêter et supprimer les volumes Redis :

```bash
docker compose down -v
```

---

## Variables d'environnement

Copiez `.env.example` en `.env` avant de lancer le projet. Ne committez jamais le `.env` (il est dans `.gitignore`).

| Variable      | Valeur par défaut          | Description                          |
|---------------|----------------------------|--------------------------------------|
| `APP_ENV`     | `development`              | Environnement applicatif             |
| `APP_VERSION` | `1.0.0`                    | Version affichée dans `/health`      |
| `REDIS_URL`   | `redis://127.0.0.1:6379`   | URL de connexion Redis               |
| `PORT`        | `3001`                     | Port d'écoute du backend             |
| `JWT_SECRET`  | `taskflow_secret_key`      | Clé de signature des tokens JWT      |

---

## API REST

### Routes publiques

| Méthode | Route            | Body                          | Description                   |
|---------|------------------|-------------------------------|-------------------------------|
| GET     | `/health`        | —                             | État de l'app et Redis        |
| GET     | `/stats`         | —                             | Statistiques par statut       |
| POST    | `/auth/register` | `{ username, password }`      | Créer un compte               |
| POST    | `/auth/login`    | `{ username, password }`      | Obtenir un token JWT          |

### Routes protégées (Bearer token requis)

| Méthode | Route          | Body                                           | Description             |
|---------|----------------|------------------------------------------------|-------------------------|
| GET     | `/tasks`       | —                                              | Lister toutes les tâches |
| POST    | `/tasks`       | `{ title, description?, priority? }`           | Créer une tâche         |
| PUT     | `/tasks/:id`   | `{ title?, description?, status?, priority? }` | Modifier une tâche      |
| DELETE  | `/tasks/:id`   | —                                              | Supprimer une tâche     |

Valeurs `status` : `todo` · `in-progress` · `done`  
Valeurs `priority` : `low` · `medium` · `high`

---

## Tests et lint

```bash
cd backend
npm test        # tests unitaires (aucune connexion Redis requise)
npm run lint    # vérification ESLint (règles : semi, quotes, no-unused-vars)
npm audit       # audit des dépendances npm
```

---

## Pipeline CI/CD

Le pipeline GitHub Actions se déclenche :
- sur chaque **push** vers `main` ou `staging`
- sur chaque **pull request** vers `main`
- sur chaque **tag** `v*` (ex. `v1.0.0`)

```
┌──────────────────────────────────────────────────────────────────┐
│                         CI/CD Pipeline                           │
│                                                                  │
│  push/PR         ┌─────────────┐   ┌──────────┐                 │
│  ──────────────► │    test     │──►│   lint   │──┐              │
│                  │ Node 18+20  │   └──────────┘  │  ┌─────────┐ │
│                  │ (parallèle) │                  ├─►│  build  │ │
│                  └─────────────┘   ┌──────────┐  │  │ (tags   │ │
│                         │          │  audit   │──┘  │  v* only)│ │
│                         └─────────►│ npm audit│     └────┬────┘ │
│                                    └──────────┘          │      │
│                                                    ┌──────┴────┐ │
│                                                    │  staging  │ │
│                                                    │ (branch)  │ │
│                                                    ├───────────┤ │
│                                                    │production │ │
│                                                    │  (tags)   │ │
│                                                    └───────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Jobs détaillés

| Job               | Déclencheur          | Description                                                                 |
|-------------------|----------------------|-----------------------------------------------------------------------------|
| `test`            | Tous les événements  | Matrice Node 18 et 20 en parallèle, Redis service inclus, `npm test`        |
| `lint`            | Après `test`         | ESLint avec Node 20, `npm run lint`                                         |
| `audit`           | Après `test`         | `npm audit --audit-level=high`, upload artifact si échec                    |
| `build`           | Tags `v*` seulement  | Build image Docker, scan Trivy (CRITICAL), push sur Docker Hub              |
| `deploy-staging`  | Branche `staging`    | `kubectl set image`, rollout status, smoke test sur `/health`               |
| `deploy-production` | Tags `v*`          | `kubectl set image` en production avec kubeconfig secret                    |

### Secrets GitHub requis

| Secret               | Utilisation                        |
|----------------------|------------------------------------|
| `DOCKERHUB_USERNAME` | Login Docker Hub                   |
| `DOCKERHUB_TOKEN`    | Token Docker Hub                   |
| `KUBECONFIG_STAGING` | Kubeconfig du cluster staging      |
| `KUBECONFIG_PRODUCTION` | Kubeconfig du cluster production |

---

## Structure du dépôt

```
TaskFlow/
├── .github/
│   └── workflows/
│       └── ci.yml              ← Pipeline CI/CD complet
├── backend/
│   ├── Dockerfile              ← Multi-stage build, user non-root
│   ├── server.js               ← API REST Node.js (sans framework)
│   ├── server.test.js          ← Tests unitaires
│   ├── eslintrc.json           ← Config ESLint
│   └── package.json
├── frontend/
│   ├── Dockerfile              ← Image Nginx Alpine
│   ├── index.html              ← Interface Kanban + Auth JWT
│   ├── app.js                  ← Logique frontend
│   ├── styles.css              ← Styles glassmorphism
│   └── nginx.conf              ← Config Nginx (gzip, cache, health)
├── k8s/
│   ├── staging/
│   │   ├── namespace.yml       ← Namespace taskflow-staging
│   │   ├── configmap.yml       ← APP_ENV=staging, PORT, REDIS_HOST
│   │   ├── secret.yml          ← Secrets encodés base64
│   │   ├── deployment.yml      ← Redis (×1), Backend (×2), Frontend (×1)
│   │   ├── service.yml         ← ClusterIP Redis, NodePort Backend/Frontend
│   │   └── ingress.yml         ← Ingress nginx (/ → frontend, /api → backend)
│   └── production/
│       ├── namespace.yml       ← Namespace taskflow-production
│       ├── configmap.yml       ← APP_ENV=production, REDIS_URL
│       ├── secret.yml          ← Secrets encodés base64
│       ├── deployment.yml      ← Redis (×1), Backend (×3 RollingUpdate), Frontend (×3)
│       └── service.yml         ← ClusterIP Redis, NodePort Backend (30001), Frontend (30081)
├── .env.example                ← Template de configuration
├── .gitignore
├── docker-compose.yml          ← Stack locale (frontend + backend + cache)
└── README.md
```

---

## Déploiement Kubernetes

### Staging

```bash
# Activer l'addon Ingress sur minikube
minikube addons enable ingress

# Appliquer tous les manifests staging
kubectl apply -f k8s/staging/

# Vérifier le déploiement
kubectl get all -n taskflow-staging

# Accès via NodePort
minikube service frontend-svc -n taskflow-staging
# ou directement :
# Frontend : http://<minikube-ip>:30082
# Backend  : http://<minikube-ip>:30002
```

### Production

```bash
# Appliquer tous les manifests production
kubectl apply -f k8s/production/

# Vérifier le déploiement
kubectl get all -n taskflow-production

# Accès via NodePort
# Frontend : http://<node-ip>:30081
# Backend  : http://<node-ip>:30001
```

### Mettre à jour l'image en production

```bash
kubectl set image deployment/taskflow-backend \
  backend=miyxki/taskflow-backend:<nouvelle-version> \
  -n taskflow-production

kubectl rollout status deployment/taskflow-backend -n taskflow-production
```

---

## Bonus

### Bonus A — Statistiques (`GET /stats`)

La route `GET /stats` retourne le nombre de tâches par statut et le taux de complétion :

```json
{
  "totalCreated": 12,
  "totalCompleted": 5,
  "byStatus": {
    "todo": 4,
    "in-progress": 3,
    "done": 5
  },
  "completionRate": "42%"
}
```

Les statistiques sont affichées dans la grille en haut de l'interface (À faire / En cours / Terminées / Total).

---

### Bonus B — Authentification JWT

Toutes les routes `/tasks` sont protégées par un token Bearer JWT (24h de validité).

**Créer un compte :**
```bash
curl -X POST http://localhost:3001/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret"}'
# → { "token": "eyJ..." }
```

**Se connecter :**
```bash
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret"}'
# → { "token": "eyJ..." }
```

**Utiliser le token :**
```bash
curl http://localhost:3001/tasks \
  -H "Authorization: Bearer eyJ..."
```

Le frontend intègre un formulaire de connexion/inscription. Le token est stocké en `localStorage` et injecté automatiquement dans chaque requête. Le bouton **Déconnexion** supprime le token.

---

### Bonus C — Prometheus + Grafana dans Kubernetes

Prometheus et Grafana sont déployés dans le namespace `monitoring` pour surveiller les pods Kubernetes.

```bash
# Déployer le stack monitoring
kubectl apply -f k8s/monitoring/

# Accéder à Grafana
kubectl port-forward svc/grafana 3000:3000 -n monitoring
# → http://localhost:3000 (admin / admin)
```

Le dashboard **TaskFlow — CPU & Mémoire** expose :
- Consommation CPU par pod (`container_cpu_usage_seconds_total`)
- Consommation mémoire par pod (`container_memory_working_set_bytes`)
- Nombre de replicas disponibles par déploiement

---

### Bonus D — Ingress nginx

L'Ingress nginx route le trafic sans exposer de ports backend directement :

| Chemin | Service cible   | Port |
|--------|-----------------|------|
| `/`    | `frontend-svc`  | 80   |
| `/api` | `backend-svc`   | 5000 |

```bash
# Activer l'addon Ingress sur minikube
minikube addons enable ingress

# Appliquer le manifest
kubectl apply -f k8s/staging/ingress.yml

# Vérifier
kubectl get ingress -n taskflow-staging
```

L'interface et l'API sont accessibles via une seule adresse IP (celle de minikube), sans numéro de port.

---

## Images Docker Hub

| Image                              | Tag     | Description          |
|------------------------------------|---------|----------------------|
| `miyxki/taskflow-backend`          | v1.0.3  | API Node.js          |
| `miyxki/taskflow-frontend`         | v1.0.5  | Interface Nginx      |

---

## Licence

Projet académique — Bachelor 3 Développement, cours DevOps 2025-2026.
