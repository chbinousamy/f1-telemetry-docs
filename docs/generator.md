# Générateur

`telemetry_generator.py` simule une vraie course (vitesse qui varie en virages, secteurs
chronométrés, classement par distance parcourue) et émet des paquets UDP dans le format binaire
exact du jeu — utilisable comme substitut au vrai F1 25 pour tester tout le pipeline.

## Cible réseau : `--ip` ou `--network` (mutuellement exclusifs)

| Option | Résultat | Cas d'usage |
|---|---|---|
| `--network local` | Cible `127.0.0.1` | Générateur lancé en process Windows, listener dans Docker (port publié) |
| `--network docker` | Cible `telemetry_listener` (DNS Compose) | Générateur *et* listener tous les deux en conteneur, sur `f1_network` |
| `--ip ADRESSE` | Cible l'adresse donnée | Cas custom (LAN, `host.docker.internal`, autre host) |

## Autres options

| Option | Défaut |
|---|---|
| `--port PORT` | 20777 |
| `--cars N` | 4 (1-22) |
| `--laps N` | 4 |
| `--speed-factor F` | 1.0 (0.5=plus lent, 2.0=plus rapide) |
| `--track-id N` | id réel F1-jeu correspondant à `--track`/`--replay` (ex. Monaco=5), ou 0 |
| `--track {generic,monaco,paul_ricard,silverstone}` | `generic` — mutuellement exclusif avec `--replay` |
| `--replay {austrian_2026,british_2026,belgian_2026,hungarian_2026}` | aucun — mutuellement exclusif avec `--track` |
| `--no-randomness` | désactivé (vitesse sans variation aléatoire) |

Variables d'environnement équivalentes (service Docker `telemetry_generator`) :
`GENERATOR_NETWORK`, `GENERATOR_IP`, `GENERATOR_PORT`, `GENERATOR_CARS`, `GENERATOR_LAPS`,
`GENERATOR_SPEED_FACTOR`, `GENERATOR_TRACK_ID`, `GENERATOR_TRACK`, `GENERATOR_REPLAY`.

## Circuits réels (`--track`)

`--track generic` (par défaut) simule un cercle avec un profil de vitesse sinusoïdal synthétique —
aucune donnée réelle. Les trois autres valeurs chargent un **vrai tracé** depuis
`tracks/<nom>.json`, construit par `fetch_track_layout.py` à partir de la télémétrie FastF1 d'un
vrai tour rapide : position réelle (X/Y), longueur réelle du circuit, et profil de vitesse réel en
fonction de la distance parcourue.

```bash
python telemetry_generator.py --network local --track monaco --cars 3 --laps 1
```

Concrètement, ça veut dire que les voitures freinent vraiment fort à l'épingle de Monaco et
roulent vraiment vite sur la ligne droite du Mistral à Paul Ricard — la courbure et le braquage
(`Steering Input`) sont dérivés de la vraie forme du circuit, pas d'un modèle synthétique.

| Circuit | Source FastF1 | Longueur | Tour de référence |
|---|---|---|---|
| `monaco` | Monaco 2023, course | 3275 m | 1:15.650 (HAM) |
| `paul_ricard` | French GP 2022, course | 5764 m | 1:35.781 (SAI) |
| `silverstone` | Silverstone 2023, course | 5801 m | 1:30.275 (VER) |

Vérifié : sur un test de 3 voitures / 2 tours, les temps au tour simulés (73-77s à Monaco, 94-95s
à Paul Ricard, 89-93s à Silverstone) collent au tour de référence à quelques % près — l'écart
vient de la variation de rythme par voiture (`skill`, ±3%) et du bruit aléatoire, pas d'un
problème de modèle.

### Régénérer/ajouter un circuit

```bash
pip install -r requirements-tracks.txt   # fastf1 -- outil ponctuel, pas une dépendance runtime
python fetch_track_layout.py --year 2023 --event Monaco --output tracks/monaco.json
```

`--event` accepte tout identifiant que FastF1 reconnaît (nom du Grand Prix, du circuit, ou de la
ville hôte).

## Rejouer une vraie course (`--replay`)

Contrairement à `--track` (simulation physique sur un vrai tracé), `--replay` **ne simule rien** :
il rejoue directement la télémétrie réelle enregistrée du vainqueur d'une vraie course — vitesse,
position, throttle/brake, rapport, régime, DRS, tout vient de la donnée FastF1, échantillonnée à
10 Hz. Un seul pilote, donc `--cars` est ignoré (forcé à 1) et `--laps` aussi (le nombre de tours
vient de la course réelle).

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

### Données stockées dans un volume Docker

Les fichiers `race_replays/*.json` (récupérés une fois via `fetch_race_telemetry.py`, un outil
ponctuel qui a besoin de FastF1 — non installé dans l'image du générateur) sont copiés dans un
**volume Docker nommé** `race_replays`, monté en lecture seule dans le conteneur
`telemetry_generator`. Ça découple les données (potentiellement volumineuses, ~10-12 Mo par
course) de l'image Docker : pas besoin de reconstruire l'image pour ajouter/mettre à jour une
course.

```bash
# 1. Récupérer une course (une fois, en local -- a besoin de fastf1)
pip install -r requirements-tracks.txt
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
python telemetry_generator.py --network local --cars 2 --laps 1
```

Grille complète, lente pour observer :
```bash
python telemetry_generator.py --network local --cars 20 --laps 10 --speed-factor 0.5
```

Dans Docker (profil demo) :
```bash
docker compose --profile demo up -d telemetry_generator
```

!!! info
    Par défaut, ce service Docker cible `host.docker.internal` — il fait donc un aller-retour réel
    par le port publié de l'hôte, exactement le chemin qu'emprunterait le vrai jeu, plutôt que le
    raccourci interne `f1_network`.

## Comportement de simulation

- Vitesse pilotée par un profil sinusoïdal (quelques "virages" par tour, entre 70 et 330 km/h)
- Secteurs chronométrés au franchissement du 1/3 et des 2/3 du tour
- Classement recalculé à chaque paquet lap, trié par (numéro de tour, distance totale)

!!! tip "Le générateur s'arrête tout seul"
    Une fois que toutes les voitures ont terminé leurs tours (ou après 20 000 ticks de sécurité) —
    pas besoin de le tuer manuellement pour un test court.
