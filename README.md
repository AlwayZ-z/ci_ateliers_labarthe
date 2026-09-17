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
