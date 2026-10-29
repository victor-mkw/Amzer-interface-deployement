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

Par défaut, la dernière version (`latest`) est utilisée. Pour imposer une version, ajouter dans le `.env` :

```
AMZER_TAG=1.0.0
```

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

## Mise en production derrière un proxy

L'application n'écoute que sur `127.0.0.1:8050`. L'accès public passe par le nginx de la machine, qui gère le HTTPS. Exemple :

```nginx
server {
    listen 443 ssl;
    server_name amzer.exemple.fr;

    location / {
        proxy_pass http://127.0.0.1:8050;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

## Mise à jour

```bash
docker compose pull
make up
```

Les utilisateurs devront se reconnecter. Pour revenir à une version précédente, fixer `AMZER_TAG` dans le `.env` puis relancer la même commande.

## En cas de problème

| Symptôme | Piste |
|---|---|
| Déconnexion à chaque redémarrage | `SECRET_KEY` absente ou vide dans le `.env`. |
| Impossible de se connecter en HTTP (test en local) | Le cookie de session exige HTTPS. Pour un essai sur `http://localhost:8050`, ajouter `SESSION_COOKIE_SECURE: "false"` et `BEHIND_PROXY: "false"` sous `environment:` dans `docker-compose.yml`, et les retirer ensuite. |
| Erreur de connexion à l'API | Vérifier `API_URL` et que l'API est démarrée et présente sur `amzer-net` (voir « Lien avec l'API »). |
| `network amzer-net declared as external, but could not be found` | `docker network create amzer-net`. |
| `denied` au téléchargement de l'image | Refaire `docker login ghcr.io` avec le token fourni. |
| Page inaccessible | `make logs` pour voir l'erreur. |
