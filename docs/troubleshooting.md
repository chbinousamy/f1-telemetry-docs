# Dépannage

## Aucun paquet reçu

!!! warning
    - Docker en cours : `docker compose ps`
    - Logs listener sans erreur : `docker compose logs telemetry_listener`
    - Générateur/jeu envoie bien vers la **bonne adresse** — voir le tableau de décision réseau dans [Vue d'ensemble](overview.md#topologie-reseau-docker)
    - Le listener conteneurisé doit toujours tourner en `--network docker` (défaut) — sinon le port publié ne sert à rien

## "127.0.0.1 ne marche pas depuis mon conteneur"

!!! info
    Normal — chaque conteneur a son propre loopback isolé, distinct de celui de l'hôte et de celui
    des autres conteneurs. Depuis un conteneur, cibler soit le nom du service Docker
    (`telemetry_listener`, résolu via `f1_network`), soit `host.docker.internal` pour atteindre
    l'hôte. `127.0.0.1` n'a de sens que si le process qui envoie tourne **directement sur l'hôte**.

## Wireshark ne montre rien

!!! warning
    Cause la plus fréquente : le générateur (ou le listener) tourne dans un conteneur — ce trafic
    ne traverse jamais une interface réseau Windows visible par Wireshark, même s'il arrive bien à
    destination (vérifiable dans MongoDB). Voir [Wireshark](wireshark.md).

## MongoDB connexion refusée

```bash
docker compose ps
docker compose logs mongodb
```

Vérifier que `mongodb` est bien `Up (healthy)` avant que le listener ne démarre (dépendance
`condition: service_healthy` dans `docker-compose.yml`).

!!! tip "Après un redémarrage de Docker Desktop"
    Seul `telemetry_listener` a `restart: unless-stopped` dans `docker-compose.yml` — pas
    `mongodb`. Si Docker Desktop redémarre, le listener peut se retrouver en boucle de
    redémarrage tant que MongoDB n'est pas relancé. Un simple `docker compose up -d` relance
    tout proprement.

## query_telemetry.py ne retourne rien

- Vérifier qu'un générateur (ou le jeu) a bien tourné récemment
- `--temps` et `--laps` lisent la collection `car_telemetry_packets` — si elle est vide, rien à afficher, c'est normal en tout début de session

## `--archive` plante avec une erreur d'encodage

Un bug déjà corrigé (caractère unicode `⚠` incompatible avec l'encodage par défaut de la console
Windows cp1252) — si ça se reproduit avec un autre message, éviter les caractères non-ASCII dans
les `print()` du script, ou forcer `PYTHONIOENCODING=utf-8`.

## Checklist rapide

!!! success
    - `docker compose ps` → mongodb healthy, telemetry_listener up
    - `docker compose logs telemetry_listener` → pas d'erreur
    - Générateur (local ou Docker) ou vrai jeu, ciblant la bonne adresse
    - `python query_telemetry.py --session` retourne des données
