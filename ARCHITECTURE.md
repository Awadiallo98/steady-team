# Architecture

> **Statut** : architecture **cible**. Le code n'existe pas encore ; ce document décrit ce qui va être construit et sera mis à jour à chaque changement de structure.

## 1. Vue d'ensemble

Team Dashboard est une **API REST** (FastAPI) qui sert un dashboard d'équipe : kanban avec limite WIP, timer par tâche, check-in bien-être anonyme et demande d'aide confidentielle.

L'API est construite **en premier** et testée via Swagger. Un front Next.js viendra ensuite : il ne fera que de l'affichage, toute la logique métier reste dans l'API.

L'application est conteneurisée avec Docker et déployée sur Azure en **PaaS** (services managés).

## 2. Schéma global

```
                         Clients
      Swagger (aujourd'hui) · front Next.js (plus tard)
                            │
                            │ HTTPS + jeton JWT
                            ▼
┌──────────── Azure : un groupe de ressources, région UE ────────────┐
│                                                                    │
│  Azure Container Apps                                              │
│  ┌───────────── API FastAPI (conteneur Docker) ──────────────┐     │
│  │   routes  ──▶  services  ──▶  repositories                │     │
│  └─────────┬──────────────────────┬──────────────────┬───────┘     │
│            │ secrets              │ logs, métriques  │ SQL         │
│            │ (identité managée)   │                  │             │
│            ▼                      ▼                  ▼             │
│       Key Vault          Application Insights   PostgreSQL managé  │
│                                                                    │
│  Container Registry : stocke les images Docker de l'API            │
└────────────────────────────────────────────────────────────────────┘
                            ▲
                            │ tests, build, push de l'image, déploiement
                            │ (connexion à Azure par OIDC, sans mot de passe)
                      GitHub Actions
```

En local, le même conteneur API tourne avec docker-compose, à côté d'un conteneur PostgreSQL, et les secrets viennent d'un fichier `.env` non versionné.

## 3. Les couches de l'API

Une requête traverse les couches **dans un seul sens** :

```
requête HTTP ──▶ route ──▶ service ──▶ repository ──▶ base PostgreSQL
```

| Couche | Rôle | Exemple |
|---|---|---|
| **Route** (`api/routes/`) | Reçoit la requête HTTP, valide l'entrée, appelle un service, renvoie la réponse. Aucune règle métier. | `POST /tasks/{id}/move` |
| **Schéma** (`schemas/`) | Décrit ce qui entre et sort de l'API (Pydantic). | `TaskMove`, `TaskOut` |
| **Service** (`services/`) | Contient les **règles métier**. | limite WIP, un seul timer actif, seuil d'anonymat |
| **Repository** (`repositories/`) | Seul endroit qui parle à la base. | `get_tasks_in_progress(user_id)` |
| **Modèle** (`models/`) | Décrit les tables de la base. | `Task`, `CheckIn`, `HelpRequest` |
| **Core** (`core/`) | Configuration, sécurité (JWT), connexion à la base. | lecture des variables d'environnement |

**Pourquoi ce découpage** : chaque couche a une seule responsabilité. Les règles métier se testent sans HTTP ni base, et on peut changer de base ou d'interface sans toucher aux règles.

## 4. Composants Azure

Tous les composants vivent dans **un seul groupe de ressources**, en **région européenne** (RGPD), pour pouvoir tout superviser et tout supprimer d'un coup.

| Composant | Rôle dans le projet | Modèle | Concept AZ-900 |
|---|---|---|---|
| Azure Container Apps | Exécute le conteneur de l'API, le duplique si la charge augmente | PaaS | Calcul, conteneurs, élasticité |
| Azure Container Registry | Stocke les images Docker construites par la CI | PaaS | Conteneurs |
| Azure Database for PostgreSQL | Base de données managée (sauvegardes, correctifs gérés par Azure) | PaaS | Bases de données managées |
| Azure Key Vault | Stocke les secrets (mot de passe de la base, clé JWT) | PaaS | Sécurité, gestion des secrets |
| Identité managée | Permet à l'API de lire Key Vault sans mot de passe | Identité | Entra ID, Zero Trust |
| Application Insights | Logs, métriques et traces de l'API | PaaS | Supervision (Azure Monitor) |
| Groupe de ressources + alerte de budget | Regroupe les ressources, surveille les coûts | Gouvernance | Gestion des coûts |

**Responsabilité partagée** : en PaaS, Azure gère le matériel, le système d'exploitation et les correctifs. Le projet reste responsable du **code, des accès et des données** : par exemple, interdire à un manager de lire un check-in individuel est une règle de notre code, pas d'Azure.

## 5. Données sensibles : qui voit quoi

| Donnée | La personne elle-même | Le manager |
|---|---|---|
| Kanban (tâches, assignations) | ses tâches | tout le tableau de son équipe |
| Temps passé | son propre détail | le temps **par tâche** uniquement |
| Check-in du matin | ses réponses | vue **agrégée et anonyme**, au-dessus d'un seuil de réponses ; **aucune route** ne renvoie un check-in individuel |
| Demande d'aide | ses demandes | seulement s'il est le **destinataire choisi** |

La **demande d'aide** est un flux séparé du check-in :
- table séparée, droits d'accès propres, conservation courte ;
- l'anonymat n'est levé que si la personne le choisit, à chaque demande ;
- elle n'entre dans aucune statistique ;
- les ressources d'urgence affichées sont fixes, jamais générées.

## 6. Sécurité en profondeur

| Niveau | Mesures |
|---|---|
| **Code** | Authentification par JWT (qui tu es) ; autorisation par rôle **et** propriété des données (ce que tu peux faire) ; un test Pytest pour chaque règle d'accès importante. |
| **Azure** | Identité managée, secrets dans Key Vault, accès réseau restreint à la base, connexion CI par OIDC. |
| **Données** | Minimisation, données de bien-être agrégées, purge automatique des données sensibles, aucune donnée nominative envoyée à un modèle d'IA. |

## 7. Décisions d'architecture

Chaque choix important est documenté dans [`docs/decisions/`](docs/decisions/) (format ADR : contexte, décision, alternatives écartées, conséquences).
