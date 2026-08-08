# Installation

## Prérequis

| Logiciel | Destination | Obligatoire |
|---|---|---|
| **Docker Desktop** (WSL2 backend sur Windows) | MongoDB, listener, générateur en conteneurs | ✅ Oui |
| **Python 3.6+** | `query_telemetry.py` / `telemetry_generator.py` lancés hors Docker | ✅ Oui |
| **Wireshark** | Analyse réseau | ❌ Optionnel |
| **F1 25** | Source de télémétrie réelle | ❌ Optionnel — le générateur en tient lieu |

## Installer Docker Desktop (Windows)

1. Télécharger : `https://www.docker.com/products/docker-desktop`
2. Lancer l'installateur, sélectionner **"WSL 2 Backend"** (recommandé)
3. Redémarrer si demandé
4. Vérifier :
   ```bash
   docker --version
   ```

## Dépendances Python

```bash
cd C:\Users\Sam\Projects\F1Telemetry
pip install -r requirements.txt
```

Installe `pymongo` et `dnspython` — nécessaires pour lancer `query_telemetry.py` ou
`telemetry_generator.py` hors Docker.

## Démarrer les conteneurs

=== "MongoDB + Listener (usage normal)"

    ```bash
    docker compose up -d --build
    ```

=== "+ Générateur de démo (tester sans le jeu)"

    ```bash
    docker compose --profile demo up -d --build
    ```

Vérifier :
```bash
docker compose ps
```
Attendu : `f1_mongodb` — Up (healthy), `f1_telemetry_listener` — Up.

Logs :
```bash
docker compose logs -f telemetry_listener
```
Attendu : `Connected to MongoDB` puis `UDP listener started on 0.0.0.0:20777`.

## Configurer F1 25 pour envoyer vers ce listener

!!! success "Simple sur Windows + Docker Desktop"
    Le port 20777 du conteneur est publié sur l'hôte (`ports: ["20777:20777/udp"]` dans
    `docker-compose.yml`) — Docker Desktop redirige automatiquement `127.0.0.1:20777` vers le
    conteneur. Pas besoin de chercher une IP WSL2 particulière : si le jeu tourne sur cette même
    machine Windows, `127.0.0.1` suffit. Vérifié en conditions réelles (voir [Wireshark](wireshark.md)).

Dans le jeu : **Réglages → Réglages du jeu → Réglages UDP Telemetry** :

| Paramètre | Valeur |
|---|---|
| UDP Telemetry | On |
| UDP Broadcast Mode | Off |
| UDP IP Address | `127.0.0.1` |
| UDP Port | `20777` |
| UDP Format | `2025` |

!!! warning "Le jeu tourne sur une autre machine ?"
    (PC séparé, console PS5/Xbox sur le même réseau) : remplacer `127.0.0.1` par l'IP LAN réelle
    de cette machine Windows, et vérifier que le pare-feu Windows autorise le trafic UDP entrant
    sur le port 20777 (le loopback contourne le pare-feu, le trafic LAN externe non).

## Tester le système

### Sans le jeu : avec le générateur

1. Lancer le générateur en local (process Windows, cible le port publié) :
   ```bash
   python telemetry_generator.py --network local --cars 2 --laps 1
   ```
2. Vérifier les logs du listener :
   ```bash
   docker compose logs -f telemetry_listener
   ```
3. Interroger les données :
   ```bash
   python query_telemetry.py --session --laps --car 0
   ```

### Avec le vrai jeu

1. Configurer le jeu comme ci-dessus, démarrer une session
2. Après ~30 secondes de jeu :
   ```bash
   python query_telemetry.py --telemetry --limit 5
   ```

## Checklist

- [ ] Docker Desktop installé et lancé
- [ ] Python 3.6+ installé
- [ ] `pip install -r requirements.txt`
- [ ] `docker compose up -d --build`
- [ ] `docker compose ps` — mongodb healthy, listener up
- [ ] `docker compose logs telemetry_listener` — pas d'erreur
- [ ] Test générateur local réussi
- [ ] `query_telemetry.py --session` retourne des données
- [ ] Jeu F1 25 configuré (UDP IP 127.0.0.1, port 20777, format 2025)

## Étapes suivantes

1. [Générateur](generator.md) pour des scénarios de test avancés
2. [Wireshark](wireshark.md) pour inspecter les paquets
3. [Requêtes](query.md) pour explorer les données
4. En cas de problème : [Dépannage](troubleshooting.md)
