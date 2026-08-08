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
| `--track-id N` | 0 |
| `--no-randomness` | désactivé (vitesse sans variation aléatoire) |

Variables d'environnement équivalentes (service Docker `telemetry_generator`) :
`GENERATOR_NETWORK`, `GENERATOR_IP`, `GENERATOR_PORT`, `GENERATOR_CARS`, `GENERATOR_LAPS`,
`GENERATOR_SPEED_FACTOR`, `GENERATOR_TRACK_ID`.

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
