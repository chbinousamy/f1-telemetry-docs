# Requêtes

`query_telemetry.py` interroge, exporte et archive les données FastF1 stockées en MongoDB.

## Commandes de base

| Commande | Résultat |
|---|---|
| `python query_telemetry.py --session` | Info session (météo, temp piste) |
| `python query_telemetry.py --laps --car 0` | Temps de tours pour la voiture 0 |
| `python query_telemetry.py --telemetry` | Derniers paquets telemetry (speed, throttle...) |
| `python query_telemetry.py --temps --car 0` | Températures pneus/freins voiture 0 |
| `python query_telemetry.py --export data.json` | Export JSON (derniers paquets car_telemetry, toutes sessions confondues) |

```bash
python query_telemetry.py --car 5 --limit 100 \
  --mongo-uri "mongodb://root:password@localhost:27017/f1_telemetry?authSource=admin"
```

## Archiver une session complète

`--archive` fige **une session entière** (les 4 types de paquets) dans `archives/<nom>/`, prête à
être versionnée avec git — contrairement à `--export` qui ne prend que les N derniers paquets
`car_telemetry`, toutes sessions mélangées.

| Option | Défaut | Description |
|---|---|---|
| `--archive` | — | Déclenche l'archivage |
| `--session-uid ID` | session la plus récente | Session à archiver |
| `--archive-name NOM` | `session_uid` brut | Nom du sous-dossier sous `archives/` |
| `--archive-no-motion` | inclus | Exclut `motion_packets` (60 Hz — souvent la plus grosse collection) |
| `--archive-no-telemetry` | inclus | Exclut `car_telemetry_packets` (60 Hz) |

```bash
python query_telemetry.py --archive --archive-name "course_spa_2026-08-08"
```

Produit :
```text
archives/course_spa_2026-08-08/
├── session.json
├── laps.json
├── telemetry.json
├── motion.json
└── summary.json          # compte de documents + taille par fichier
```

Puis, pour l'envoyer sur GitHub :
```bash
git add archives/course_spa_2026-08-08
git commit -m "Archive course_spa_2026-08-08"
git push
```

!!! warning "Volumétrie"
    `motion_packets` et `car_telemetry_packets` tournent à 60 Hz — une session de quelques minutes
    à grille complète peut représenter plusieurs dizaines de Mo. Le script affiche un avertissement
    par fichier au-delà de 20 Mo ; utiliser `--archive-no-motion`/`--archive-no-telemetry` pour ne
    garder que les temps de tour et les infos de session si le volume n'est pas nécessaire.

## MongoDB shell direct

```bash
docker compose exec mongodb mongosh -u root -p password --authenticationDatabase admin
```

```javascript
use f1_telemetry
db.car_telemetry_packets.countDocuments()
db.car_telemetry_packets.find().sort({_id:-1}).limit(1).pretty()
```
