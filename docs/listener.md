# Listener

`f1_telemetry_listener.py` — écoute en UDP, parse le format binaire F1 25 exact, convertit en
FastF1 et stocke en MongoDB.

## Fonctionnement

- Écoute en UDP (port 20777 par défaut)
- Parse les paquets binaires F1 25 avec le layout défini dans `f1_packet_format.py`
- Convertit chaque paquet en document FastF1 **au moment du stockage** (`store_telemetry`)
- Écrit dans MongoDB, une collection par type de paquet

## Options en ligne de commande

| Option | Défaut | Description |
|---|---|---|
| `--network {docker,local}` | `docker` | `docker` bind `0.0.0.0` (obligatoire dans un conteneur). `local` bind `127.0.0.1`, n'accepte que le trafic de cette machine (hors conteneur uniquement). |
| `--port PORT` | 20777 | Port UDP d'écoute |

Variables d'environnement équivalentes : `LISTENER_NETWORK`, `LISTENER_PORT`, `MONGO_URI`.

!!! danger "À l'intérieur d'un conteneur, toujours `--network docker`"
    Testé et confirmé : un listener en conteneur qui bind `127.0.0.1` ne reçoit **jamais** le
    trafic redirigé depuis le port publié sur l'hôte, même si ce port est bien mappé. Docker
    réinjecte ce trafic via l'interface réseau externe du conteneur, pas via son loopback interne.
    Voir [Vue d'ensemble](overview.md#topologie-reseau-docker).

## Logs à surveiller

```bash
docker compose logs -f telemetry_listener
```

- `Connected to MongoDB` — connexion OK
- `UDP listener started on 0.0.0.0:20777` — écoute active
- Toute ligne `ERROR` — parsing cassé, à investiguer immédiatement

## Redémarrer / déboguer

```bash
docker compose restart telemetry_listener
docker compose exec telemetry_listener sh
```
