# Vue d'ensemble

## Architecture système

```text
╔════════════════════════════════════════════════════════════╗
║   F1 25 (vrai jeu, host Windows) OU générateur de test      ║
║              UDP, format binaire officiel F1 25              ║
╚══════════════════════════╤═════════════════════════════════╝
                           │ port 20777
                           ▼
        ┌──────────────────────────────────────┐
        │   f1_telemetry_listener.py           │
        │  • Parse le format F1 25 exact        │
        │    (f1_packet_format.py, partagé      │
        │    avec le générateur)                │
        │  • Convertit en JSON FastF1           │
        │    au moment du stockage              │
        └──────────────────────────────────────┘
                           │ documents JSON
                           ▼
        ┌──────────────────────────────────────┐
        │        MongoDB (mongo:7.0)           │
        │  • motion_packets                     │
        │  • car_telemetry_packets              │
        │  • session_packets                    │
        │  • lap_packets                        │
        └──────────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         query_telemetry.py   mongosh   archives/ (git)
```

## Composants principaux

| Composant | Fichier | Rôle |
|---|---|---|
| UDP Listener | `f1_telemetry_listener.py` | Parse F1 25, convertit en FastF1, stocke en MongoDB |
| Layout binaire | `f1_packet_format.py` | Source de vérité unique du format (header, motion, telemetry, lap, préfixe session) |
| MongoDB | — | Stocke la télémétrie, une collection par type de paquet |
| Générateur | `telemetry_generator.py` | Rejoue une vraie course (FastF1) dans le format binaire réel du jeu |
| Wireshark | `wireshark_f1_25_dissector.lua` | Décode les paquets — ne voit que le trafic passant par une vraie interface Windows |
| Query Tool | `query_telemetry.py` | Interroge, exporte et archive MongoDB |

## Collections MongoDB & tailles de paquets

| Collection | Type Paquet | Taille totale du paquet UDP |
|---|---|---|
| `motion_packets` | Motion (id 0) | 29 (header) + 60 × 22 voitures = 1349 bytes |
| `car_telemetry_packets` | Car Telemetry (id 6) | 29 + 60 × 22 + 3 (footer) = 1352 bytes |
| `lap_packets` | Lap Data (id 2) | 29 + 57 × 22 + 2 (footer) = 1285 bytes |
| `session_packets` | Session (id 1) | 29 + 19 (préfixe seulement, voir [Format Données](data-format.md)) = 48 bytes |

## Topologie réseau Docker

C'est la partie la plus piégeuse du projet — trois `127.0.0.1` différents coexistent selon d'où
on parle, et ça détermine tout le reste.

```text
                    Machine Windows (l'hôte)
      ┌─────────────────────────────────────────────┐
      │  127.0.0.1 (hôte) ◄── vrai jeu F1 25,        │
      │       │              générateur --network    │
      │       │              local, Wireshark         │
      │       │ port publié 20777/udp                │
      │       ▼                                       │
      │  ┌─────────────── f1_network (bridge) ──────┐│
      │  │                                            ││
      │  │  [telemetry_listener]  [mongodb]           ││
      │  │   127.0.0.1 propre     127.0.0.1 propre    ││
      │  │   bind 0.0.0.0 ◄──┐    (isolé)              ││
      │  │   IP interne:     │                         ││
      │  │   172.18.0.x      │                         ││
      │  │                   │                         ││
      │  │  [telemetry_generator]                      ││
      │  │   127.0.0.1 propre (≠ hôte, ≠ listener)     ││
      │  │   cible: "telemetry_listener" (DNS)         ││
      │  │   OU "host.docker.internal" (repasse par    ││
      │  │   le port publié de l'hôte)                 ││
      │  └──────────────────────────────────────────┘│
      └─────────────────────────────────────────────┘
```

| Qui envoie | Cible | Arrive au listener ? |
|---|---|---|
| Process Windows (jeu ou générateur `--network local`) | `127.0.0.1:20777` | ✅ via le port publié |
| Conteneur (générateur `--network docker`) | `127.0.0.1:20777` | ❌ reste dans son propre conteneur |
| Conteneur (générateur `--network docker`) | `telemetry_listener:20777` | ✅ via le DNS interne `f1_network` |
| Conteneur (générateur, cible `host.docker.internal`) | `host.docker.internal:20777` | ✅ repasse par le port publié de l'hôte (Docker Desktop uniquement) |

!!! danger "Règle d'or"
    Le listener conteneurisé doit toujours écouter en `--network docker` (bind `0.0.0.0`, la
    valeur par défaut) — c'est ce qui le rend joignable à la fois depuis `f1_network` et depuis le
    port publié sur l'hôte. Le lancer en `--network local` à l'intérieur d'un conteneur casse
    silencieusement le port publié.

## Authentification MongoDB

!!! info
    - Username : `root`
    - Password : voir `.env` (ne pas committer)
    - Auth DB : `admin`
    - URI interne (réseau Docker) : `mongodb://root:password@mongodb:27017/f1_telemetry?authSource=admin`
    - URI depuis l'hôte : remplacer `mongodb` par `localhost` (port 27017 publié)

## Flux alternatifs

**Mode test (générateur local, visible dans Wireshark) :**
```text
generator (host, --network local) → 127.0.0.1:20777 → listener (Docker) → MongoDB → query_telemetry.py
```

**Mode test (générateur Docker, chemin réaliste) :**
```text
generator (Docker, host.docker.internal) → port publié hôte → NAT Docker → listener (Docker) → MongoDB
```

**Mode production (vrai jeu) :**
```text
F1 25 (host Windows, UDP IP = 127.0.0.1, port 20777) → listener (Docker) → MongoDB → query_telemetry.py
```
