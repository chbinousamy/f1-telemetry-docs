# Générateur

`telemetry_generator.py` rejoue directement la télémétrie réelle enregistrée du vainqueur d'une
vraie course F1 25 — vitesse, position, throttle/brake, rapport, régime, DRS, météo (pluie,
température air/piste), tout vient de la donnée FastF1, échantillonnée à 10 Hz (météo à sa
résolution native, ~1/min) — et émet des paquets UDP dans le format binaire exact du jeu. Aucune
simulation physique : c'est un rejeu, pas un modèle.

Non repris (voir [Format Données](data-format.md)) : le type de gomme et l'âge des pneus existent
dans FastF1 mais nécessiteraient le paquet Car Status, jamais implémenté dans
`f1_packet_format.py`.

## Cible réseau : `--ip` ou `--network` (mutuellement exclusifs)

| Option | Résultat | Cas d'usage |
|---|---|---|
| `--network local` | Cible `127.0.0.1` | Générateur lancé en process Windows, listener dans Docker (port publié) |
| `--network docker` | Cible `telemetry_listener` (DNS Compose) | Générateur *et* listener tous les deux en conteneur, sur `f1_network` |
| `--ip ADRESSE` | Cible l'adresse donnée | Cas custom (LAN, `host.docker.internal`, autre host) |

## Autres options

| Option | Défaut |
|---|---|
| `--replay {austrian_2026,british_2026,belgian_2026,hungarian_2026}` | **obligatoire** |
| `--port PORT` | 20777 |
| `--speed-factor F` | 1.0 (0.5=plus lent, 2.0=plus rapide) |
| `--track-id N` | id réel F1-jeu enregistré avec `--replay` (ex. Austria=17) |

Variables d'environnement équivalentes (service Docker `telemetry_generator`) :
`GENERATOR_NETWORK`, `GENERATOR_IP`, `GENERATOR_PORT`, `GENERATOR_SPEED_FACTOR`,
`GENERATOR_TRACK_ID`, `GENERATOR_REPLAY`.

## Rejouer une vraie course

Un seul pilote (le vainqueur) est rejoué — pas de grille complète, pas de `--cars`/`--laps` : le
nombre de tours vient directement de la course réelle.

```bash
python telemetry_generator.py --network local --replay austrian_2026
```

| Course | Vainqueur | Tours | Durée réelle |
|---|---|---|---|
| `austrian_2026` | George Russell (Mercedes) | 71 | 86,6 min |
| `british_2026` | Charles Leclerc (Ferrari) | 52 | 87,2 min |
| `belgian_2026` | Kimi Antonelli (Mercedes) | 44 | 84,7 min |
| `hungarian_2026` | Lando Norris (McLaren) | 70 | 99,9 min |

À `--speed-factor 1.0` (défaut), une course dure donc jusqu'à 100 minutes en temps réel — utiliser
`--speed-factor` pour accélérer (`--speed-factor 100` rejoue la course en moins d'une minute, utile
pour tester le pipeline rapidement).

La direction (`Steering Input`) et la courbure (G latérale) sont dérivées des positions réelles
consécutives enregistrées — comme pour un tracé, mais point par point plutôt que par forme globale.

## Données stockées dans un volume Docker

Les fichiers `race_replays/*.json` (récupérés une fois via `fetch_race_telemetry.py`, un outil
ponctuel qui a besoin de FastF1 — non installé dans l'image du générateur) sont copiés dans un
**volume Docker nommé** `race_replays`, monté en lecture seule dans le conteneur
`telemetry_generator`. Ça découple les données (potentiellement volumineuses, ~10-12 Mo par
course) de l'image Docker : pas besoin de reconstruire l'image pour ajouter/mettre à jour une
course.

```bash
# 1. Récupérer une course (une fois, en local -- a besoin de fastf1)
pip install -r requirements-replay.txt
python fetch_race_telemetry.py --year 2026 --event Austria --track-id 17 \
  --output race_replays/austrian_2026.json

# 2. Copier dans le volume Docker nommé (une fois, ou après mise à jour)
docker compose --profile seed run --rm seed_race_replays

# 3. Utiliser depuis le conteneur (le volume est déjà monté)
docker compose --profile demo run --rm -e GENERATOR_REPLAY=austrian_2026 telemetry_generator
```

En local (hors Docker), `--replay` lit directement `./race_replays/<nom>.json` — pas besoin du
volume ni de l'étape 2.

!!! success "Vérifié"
    Testé de bout en bout dans le vrai conteneur (volume monté, variable `GENERATOR_REPLAY`) :
    71 tours rejoués sans erreur, `track_id` correctement à 17 (Austria) dans MongoDB, temps au
    tour réels préservés dans les paquets lap.

## Exemples

Test rapide en local (visible dans Wireshark) :
```bash
python telemetry_generator.py --network local --replay hungarian_2026 --speed-factor 200
```

Dans Docker (profil demo, course définie par `GENERATOR_REPLAY` dans `docker-compose.yml`) :
```bash
docker compose --profile demo up -d telemetry_generator
```

!!! info
    Par défaut, ce service Docker cible `host.docker.internal` — il fait donc un aller-retour réel
    par le port publié de l'hôte, exactement le chemin qu'emprunterait le vrai jeu, plutôt que le
    raccourci interne `f1_network`.

!!! tip "Le générateur s'arrête tout seul"
    Une fois la course terminée (ou après 120 000 ticks de sécurité, largement au-delà de la durée
    de la plus longue course) — pas besoin de le tuer manuellement.
