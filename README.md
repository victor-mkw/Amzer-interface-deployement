# Amzer Interface — Application web

Interface web d'Amzer : feuilles de temps, calendrier, projets, synthèse et gestion. Elle est livrée sous forme d'image Docker et se connecte à l'API Amzer.

## Démarrage rapide

```bash
cp .env.exemple .env                   # puis renseigner les valeurs (voir ci-dessous)
docker login ghcr.io -u <utilisateur>  # avec le token d'accès fourni, voir « Image Docker »
docker network create amzer-net        # une seule fois, voir « Lien avec l'API »
make up                                # tire l'image et démarre l'application
```

L'application est disponible sur `http://localhost:8050`. L'API Amzer doit être démarrée au préalable (elle est livrée à part) et rattachée au réseau `amzer-net`.

## Configuration (`.env`)

| Variable | Rôle |
|---|---|
| `API_URL` | Adresse de l'API, **obligatoire**. Avec le réseau partagé : `http://amzer-api:8000/api/v1`. |
| `SECRET_KEY` | Clé secrète des sessions, **obligatoire en production**. Sans elle, les utilisateurs sont déconnectés à chaque redémarrage. Pour en générer une : `python -c "import secrets; print(secrets.token_hex(32))"` |
| `DASH_DEBUG` | Doit rester à `false` en production. |
| `PORT` | Port d'écoute (`8050` par défaut). |
| `SESSION_IDLE_TTL` | Durée d'inactivité avant déconnexion, en secondes (`28800` = 8 h par défaut). |
| `STATIC_DATA_TTL` | Fréquence de rechargement des projets et tâches, en secondes (`300` par défaut). |

Les trois dernières sont facultatives.

## Commandes

```bash
make up       # démarrer
make logs     # suivre les logs
make down     # arrêter
```

Un redémarrage de l'application déconnecte tous les utilisateurs, qui devront se reconnecter.

## Image Docker

L'image `ghcr.io/victor-mkw/amzer-interface` est privée. Son accès demande le token en lecture seule (`read:packages`) fourni par l'équipe. Elle ne contient aucun secret : la configuration vient uniquement du `.env`.

```bash
echo "$TOKEN" | docker login ghcr.io -u <utilisateur> --password-stdin
```

Par défaut, la dernière version (`latest`) est utilisée.

## Lien avec l'API

L'application et l'API tournent chacune dans leur propre Compose et communiquent par un réseau Docker partagé, `amzer-net`.

```bash
docker network create amzer-net     # une seule fois sur la machine
```

Vérifier que les deux y sont rattachées :

```bash
docker network inspect amzer-net --format '{{range .Containers}}{{.Name}} {{end}}'
```

La liste doit contenir l'API et l'application.

| Où tourne l'API | `API_URL` |
|---|---|
| Conteneur sur `amzer-net` | `http://amzer-api:8000/api/v1` |
| Même machine, hors Docker | `http://localhost:8000/api/v1` |
| Derrière le proxy de l'IT | URL publique HTTPS fournie par l'IT |
## Mise à jour

```bash
docker compose pull
make up
```


