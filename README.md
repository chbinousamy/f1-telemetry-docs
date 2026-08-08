# f1-telemetry-docs

Sources Markdown de la documentation de [chbinousamy/f1-telemetry](https://github.com/chbinousamy/f1-telemetry),
construites avec [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) et déployées
automatiquement sur GitHub Pages à chaque push sur `master`.

Site publié : **https://chbinousamy.github.io/f1-telemetry-docs/**

## Aperçu local

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

## Déploiement

Géré par `.github/workflows/deploy.yml` (`mkdocs gh-deploy --force` sur chaque push). Pas besoin
de lancer `mkdocs gh-deploy` manuellement.
