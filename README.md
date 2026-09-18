# ci_ateliers_labarthe

[![CI](https://github.com/AlwayZ-z/ci_ateliers_labarthe/actions/workflows/ci.yml/badge.svg)](https://github.com/AlwayZ-z/ci_ateliers_labarthe/actions/workflows/ci.yml)

Application Flask minimale servant de support a l'atelier note "Pipeline CI avec GitHub Actions" (ESIEA, bloc DevOps).

## Endpoints

- `GET /health` : retourne `{"status": "ok"}`
- `GET /status` : retourne le nom et la version du service

## Ce que fait la CI

Le workflow `.github/workflows/ci.yml` se declenche a chaque push sur `main` et a chaque pull request.

Il enchaine deux etages :

1. **lint** : verifie le respect de la PEP8 avec `flake8`, configure dans `.flake8` (ligne max 100 caracteres).
2. **test** : ne demarre que si le lint est passe (`needs: lint`). Execute `pytest` sur une matrice de trois versions de Python (3.10, 3.11, 3.12) et produit un rapport de couverture.

Les dependances pip sont mises en cache entre les executions (cle basee sur le hash de `requirements.txt`), et le rapport de couverture HTML est conserve comme artefact telechargeable pendant 7 jours, y compris lorsque les tests echouent.

La branche `main` est protegee : une pull request est obligatoire et les quatre checks (`lint`, `test (3.10)`, `test (3.11)`, `test (3.12)`) doivent etre au vert avant tout merge.

## Lancer en local

```bash
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
flake8 .
pytest -v
```
## Conteneurisation Docker (séance 3)

### Construire l'image

```bash
docker build -t app-slim:1.0 .
```

Le `Dockerfile` est un multi-stage. Un stage `builder`, basé sur `python:3.12`, installe les
dépendances dans `/install`. Le stage final, basé sur `python:3.12-slim`, ne récupère que ce
dossier via `COPY --from=builder` : l'image livrée ne contient donc ni cache pip, ni outils de
build. Le serveur est `gunicorn` (2 workers), pas le serveur de développement Flask.

Le conteneur tourne sous l'utilisateur non privilégié `appuser` :

```bash
docker run --rm app-slim:1.0 whoami   # appuser
```

Un `.dockerignore` limite le contexte de build : il est passé de plusieurs mégaoctets à 2,67 ko.

### Gain de taille mesuré

| Image | Dockerfile | Taille disque | Taille de contenu |
|---|---|---|---|
| `app-naive:1.0` | `Dockerfile.naive` — mono-stage, `python:3.12` | 1,64 Go | 423 Mo |
| `app-slim:1.0` | `Dockerfile` — multi-stage, `python:3.12-slim` | 212 Mo | 51,3 Mo |

Soit un facteur **8,2** sur la taille de contenu. Mesuré avec `docker images` après
reconstruction des deux images. `docker history` montre que l'écart vient presque entièrement
de l'image de base : les couches applicatives ne pèsent que 33,4 Mo.

### Lancer la stack complète

```bash
docker compose up -d --build
```

Deux services sur le réseau dédié `appnet` : `web` (cette image) et `redis` (`redis:7-alpine`).
Les données Redis vivent dans le volume nommé `redis-data` avec l'append-only activé, ce qui
fait survivre le compteur de `/visits` à un redémarrage du conteneur `web`. Le service `web`
ne démarre qu'une fois Redis réellement prêt (`depends_on` avec `condition: service_healthy`),
et son `HEALTHCHECK` interroge `/health` en Python — pas en `curl`, absent de l'image slim.

```bash
docker compose ps   # les deux services passent à (healthy) après quelques secondes
```

### Endpoints

- `GET /health` — état du service
- `GET /status` — nom et version
- `GET /visits` — compteur de visites, persistant dans Redis

### Image publiée

`ghcr.io/alwayz-z/ci-ateliers-labarthe-web`, taguée `1.0.0` et `latest`.

```bash
docker pull ghcr.io/alwayz-z/ci-ateliers-labarthe-web:1.0.0
```

Le package est public : le `pull` fonctionne sans authentification.