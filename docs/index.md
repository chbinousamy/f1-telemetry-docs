# F1 25 Telemetry Collector

<p align="center">
  <img src="img/f1-logo.png" alt="F1 25 Telemetry Collector" width="420">
</p>

Capture la télémétrie UDP du jeu **F1 25** (format binaire officiel), la convertit au format
**FastF1** et la stocke dans **MongoDB**. Inclut un générateur de télémétrie simulée pour tester
sans le jeu, et un dissecteur Wireshark pour inspecter les paquets.

!!! info "Code source"
    Ce site documente le projet [chbinousamy/f1-telemetry](https://github.com/chbinousamy/f1-telemetry)
    (listener, générateur, scripts). Les sources de *cette documentation* vivent séparément dans
    [chbinousamy/f1-telemetry-docs](https://github.com/chbinousamy/f1-telemetry-docs).

## Caractéristiques

- Listener UDP parsant le format binaire **officiel** F1 25 (header 29 bytes, motion/telemetry/lap/session)
- Layout binaire défini une seule fois (`f1_packet_format.py`), partagé entre listener et générateur — ils ne peuvent plus diverger
- Conversion FastF1 au moment du stockage MongoDB
- Générateur de simulation réaliste (vitesse en virage, secteurs, classement par distance)
- `--network {docker,local}` sur listener et générateur pour gérer proprement le passage Docker ↔ hôte
- Dissecteur Wireshark dédié pour décoder les paquets en clair
- `query_telemetry.py --archive` pour figer une session complète en JSON, prête à versionner

## Démarrage rapide

```bash
# MongoDB + listener
docker compose up -d --build

# Tester sans le jeu (process local, cible le port publié)
python telemetry_generator.py --network local --cars 4 --laps 4

# Vérifier les données
python query_telemetry.py --session
```

Toutes les options du générateur (dont `--track` pour un vrai circuit) : [Générateur](generator.md).

Pour le vrai jeu : **Réglages → Réglages UDP Telemetry**, avec `UDP IP Address = 127.0.0.1`,
`UDP Port = 20777`, `UDP Format = 2025`. Détails dans [Installation](installation.md).

!!! warning "Le réseau Docker est le piège n°1 de ce projet"
    `127.0.0.1` ne pointe pas vers la même chose selon qu'on est dans un conteneur ou sur l'hôte.
    Trois `127.0.0.1` différents coexistent (hôte, conteneur listener, conteneur générateur). Lire
    [Vue d'ensemble](overview.md) avant de déboguer un problème de connectivité.

## Cas d'usage

| Cas d'usage | Commande |
|---|---|
| Capturer le vrai jeu F1 25 | `docker compose up -d` + configurer le jeu |
| Tester sans le jeu | `python telemetry_generator.py --network local` |
| Tester le chemin réseau complet en Docker | `docker compose --profile demo up -d` (cible `host.docker.internal`) |
| Analyser les paquets réseau | Voir [Wireshark](wireshark.md) |
| Exporter/archiver une session | `python query_telemetry.py --archive` |
| Simuler un vrai circuit (Monaco, Paul Ricard, Silverstone) | `python telemetry_generator.py --network local --track monaco` |
| Analyser une session dans MoTeC i2 Pro | `python motec_export.py --archive archives/<nom> --car 0` |

## Fichiers du projet

```text
f1_packet_format.py            # Layout binaire F1 25 (source de vérité)
f1_telemetry_listener.py       # Listener UDP → FastF1 → MongoDB
telemetry_generator.py         # Générateur de télémétrie de test
fetch_track_layout.py          # Outil ponctuel : FastF1 → tracks/<nom>.json
tracks/                        # Tracés réels (monaco, paul_ricard, silverstone)
motec_export.py                # Convertit une session archivée en .ld MoTeC i2 Pro
query_telemetry.py             # Requêtes, export et archivage
wireshark_f1_25_dissector.lua  # Dissecteur Wireshark (copier dans %APPDATA%\Wireshark\plugins\)
docker-compose.yml             # mongodb + telemetry_listener (+ telemetry_generator via --profile demo)
mkdocs.yml                     # Config de ce site de documentation
docs/                          # Sources Markdown de ce site
archives/                      # Sessions archivées en JSON (git)
```

## Prérequis

- **Docker Desktop** (backend WSL2 sur Windows)
- **Python 3.6+** pour lancer les scripts hors conteneur
- **Wireshark** (optionnel)
- **F1 25** (optionnel — le générateur en tient lieu)
