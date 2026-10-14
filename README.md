# Team Dashboard

API d'un dashboard d'équipe : kanban, suivi du temps par tâche, check-in bien-être anonyme et demande d'aide confidentielle.

Le projet est développé **API d'abord** : le backend est testable via Swagger, l'interface web viendra ensuite et consommera cette API.

## Principes

- **API-first** : toute la logique métier vit dans l'API FastAPI ; le front ne fait que de l'affichage.
- **Bien-être protégé** : les check-ins ne sont visibles qu'en vue agrégée et anonyme, au-dessus d'un nombre minimum de réponses. Aucune route ne renvoie un check-in individuel à un manager.
- **Demande d'aide volontaire** : la personne choisit de lever son anonymat et choisit son destinataire. Rien n'est partagé par défaut.
- **Minimisation des données** : conservation courte et accès restreint pour les données sensibles.
- **Aucun secret dans le code** : configuration par variables d'environnement, puis coffre de secrets dans le cloud.

## Stack

| Domaine | Choix |
|---|---|
| Backend | Python, FastAPI (asynchrone) |
| Base de données | PostgreSQL |
| Tests | Pytest |
| Conteneurs | Docker, docker-compose |
| Cloud | Azure : Container Apps, Container Registry, Database for PostgreSQL, Key Vault, Application Insights |
| CI/CD | GitHub Actions |

## État d'avancement

- [ ] **Étape 1 : API MVP et déploiement Azure**
  - [ ] Squelette FastAPI, Docker, route `/health`
  - [ ] Kanban (colonnes, assignation, limite WIP par personne)
  - [ ] Timer par tâche (un seul timer actif par personne)
  - [ ] Check-in du matin (vue agrégée et anonyme)
  - [ ] Demande d'aide (flux séparé, destinataire choisi)
  - [ ] Déploiement Azure, secrets, supervision, CI/CD
- [ ] **Étape 2 : fonctionnalités IA** (Azure OpenAI)
- [ ] **Étape 3 : interface web** (React, Next.js)

## Documentation

- [Architecture](ARCHITECTURE.md) : vue d'ensemble et rôle des composants
- [Décisions d'architecture](docs/decisions/) : chaque choix important, ses raisons et les alternatives écartées
