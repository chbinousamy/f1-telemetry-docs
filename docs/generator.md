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
| `--track-id N` | 0 (identifiant cosmétique envoyé dans le paquet session) |
| `--track {generic,monaco,paul_ricard,silverstone}` | `generic` |
| `--no-randomness` | désactivé (vitesse sans variation aléatoire) |

Variables d'environnement équivalentes (service Docker `telemetry_generator`) :
`GENERATOR_NETWORK`, `GENERATOR_IP`, `GENERATOR_PORT`, `GENERATOR_CARS`, `GENERATOR_LAPS`,
`GENERATOR_SPEED_FACTOR`, `GENERATOR_TRACK_ID`, `GENERATOR_TRACK`.

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
