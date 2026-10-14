# 0001. Choix de FastAPI pour l'API

- **Statut** : accepté
- **Date** : 2026-10-14

## Contexte

Le projet est une **API d'abord** : elle est testée via Swagger, puis consommée par un front Next.js qui ne contient aucune logique métier.

Besoins identifiés :
- beaucoup d'opérations qui **attendent** une réponse extérieure : la base PostgreSQL, puis Azure OpenAI (étape IA) ;
- une **validation stricte** des données qui entrent et sortent, y compris du JSON produit par un modèle d'IA ;
- une **documentation d'API** à jour pour tester sans front ;
- un déploiement en **conteneurs** sur Azure Container Apps ;
- une **architecture en couches** (routes, services, repositories) conçue et maîtrisée par l'équipe.

## Décision

Utiliser **FastAPI** (Python) pour l'API.

Raisons :
1. **Asynchrone natif** : pendant qu'une requête attend la base ou Azure OpenAI, le serveur peut en traiter d'autres.
2. **Pydantic intégré** : les schémas valident automatiquement les entrées et les sorties. Le même outil servira à valider le JSON renvoyé par l'IA.
3. **Documentation OpenAPI et Swagger générées automatiquement** à partir du code : idéal pour une approche API-first.
4. **Framework léger et peu imposant** : il laisse concevoir la structure en couches décrite dans [ARCHITECTURE.md](../../ARCHITECTURE.md).

## Alternatives écartées

### Django + Django REST Framework
- **Atouts** : très complet (ORM, migrations, interface d'administration, authentification intégrés), très répandu.
- **Pourquoi écarté** : plus lourd pour une API seule, et ses conventions imposent une structure, alors que l'objectif est de concevoir et maîtriser soi-même les couches. Une grande partie de ce qu'il fournit (templates, admin) ne servirait pas.
- **Précision** : Django sait aussi traiter plusieurs requêtes en parallèle (plusieurs workers, support de l'asynchrone). La performance brute n'est donc **pas** l'argument décisif.

### Flask
- **Atouts** : léger et simple.
- **Pourquoi écarté** : pas de validation ni de documentation automatique sans extensions, et l'asynchrone y est moins naturel que dans FastAPI.

## Conséquences

**Positives**
- Swagger disponible dès la première route, sans travail supplémentaire.
- Validation des données cohérente dans toute l'API grâce à Pydantic.
- Structure du projet choisie et expliquée, pas imposée.

**Négatives et points d'attention**
- Pas d'ORM ni d'outil de migrations fournis : il faut les choisir (décision à prendre dans un prochain ADR).
- Pas d'interface d'administration : les données se consultent via l'API ou directement en base.
- Plus de décisions de structure à prendre soi-même : d'où l'importance de tenir `ARCHITECTURE.md` et les ADR à jour.

**Remarque sur la scalabilité**
La capacité à absorber plus de charge ne vient pas principalement du framework. Elle vient d'une **API sans état** (aucune donnée de session gardée en mémoire du serveur), déployée dans des **conteneurs** qu'Azure Container Apps peut dupliquer selon la charge.
