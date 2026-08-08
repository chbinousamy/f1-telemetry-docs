# Format des données

Layout binaire exact des paquets F1 25, et leur conversion en format FastF1.

!!! info "Source de vérité"
    Ce layout est défini une seule fois dans `f1_packet_format.py` à la racine du projet, et
    importé à la fois par `f1_telemetry_listener.py` (parsing) et `telemetry_generator.py`
    (émission), pour que les deux ne puissent jamais diverger. Cette page en est la documentation,
    pas une spécification indépendante — en cas de doute, le fichier Python fait foi.

!!! danger "Historique"
    Une version précédente de cette page décrivait un format maison simplifié (header 24 bytes)
    qui ne correspondait ni au format réel du jeu F1 25, ni à ce que le code parsait réellement —
    le header avait une taille de buffer incohérente avec son propre format `struct`, ce qui
    faisait planter le parsing sur **chaque** paquet reçu. Rien n'était jamais stocké. Le format
    ci-dessous est le format binaire officiel du jeu (aligné sur la spécification
    EA/Codemasters "Data Output from F1 25"), vérifié par des tests round-trip (génération →
    parsing → conversion FastF1) sans erreur.

## Architecture des paquets

F1 25 envoie sa télémétrie via UDP en paquets binaires little-endian, chacun précédé d'un header
fixe identifiant son type, suivi de données spécifiques au type de paquet.

### Header universel (29 bytes)

| Bytes | Champ | Type | Description |
|---|---|---|---|
| 0-1 | packet_format | uint16 | Version du format (2025) |
| 2 | game_year | uint8 | Année du jeu (25) |
| 3 | game_major_version | uint8 | Version majeure du jeu |
| 4 | game_minor_version | uint8 | Version mineure du jeu |
| 5 | packet_version | uint8 | Version du paquet |
| 6 | **packet_id** | uint8 | **Type de paquet** (voir ci-dessous) |
| 7-14 | session_uid | uint64 | Identifiant unique de session |
| 15-18 | session_time | float | Temps de session écoulé (secondes) |
| 19-22 | frame_identifier | uint32 | Numéro de frame |
| 23-26 | overall_frame_identifier | uint32 | Frame globale (multi-session) |
| 27 | player_car_index | uint8 | Index voiture du joueur |
| 28 | secondary_player_car_index | uint8 | Second joueur (255 = aucun) |

`packet_id` reste à l'offset 6 — pratique pour un filtre Wireshark rapide type
`udp.port==20777 && data[6]==0` sans dissecteur.

### Types de paquets gérés par ce projet

| ID | Type | Fréquence réelle du jeu | Taille par voiture |
|---|---|---|---|
| 0 | Motion | 60 Hz | 60 bytes |
| 1 | Session | 1-2 Hz | N/A (paquet unique) |
| 2 | Lap Data | 2 Hz | 57 bytes |
| 6 | Car Telemetry | 60 Hz (ou config joueur) | 60 bytes |

Le jeu émet aussi d'autres types de paquets (Event, Participants, Car Setups, Car Status, Final
Classification, Lobby Info, Car Damage, Session History) — ce projet les ignore silencieusement
(`process_packet` log un `DEBUG` et retourne).

## Motion Data (packet_id = 0)

Structure par voiture (60 bytes), jusqu'à 22 voitures après le header :

| Offset | Champ | Type | Unité |
|---|---|---|---|
| 0-11 | world_position (x,y,z) | float × 3 | mètres |
| 12-23 | world_velocity (x,y,z) | float × 3 | m/s |
| 24-29 | world_forward_dir (x,y,z) | int16 × 3 | normalisé ×32767 |
| 30-35 | world_right_dir (x,y,z) | int16 × 3 | normalisé ×32767 |
| 36-47 | g_force (lateral, longitudinal, vertical) | float × 3 | G |
| 48-59 | yaw, pitch, roll | float × 3 | radians |

!!! note
    Contrairement aux forces G, les vecteurs direction (`forward_dir`, `right_dir`) sont encodés
    en `int16` normalisés, pas en float — diviser par 32767 pour obtenir une composante entre
    -1.0 et 1.0.

### Conversion FastF1 (motion_packets)

```json
{
  "packet_type": "motion",
  "session_uid": "12cca41615ca1203",
  "frame": 1234,
  "players": [{
    "car_index": 0,
    "position": {"x": 500.5, "y": 0.5, "z": 120.0},
    "velocity": {"x": 70.2, "y": 0.0, "z": -12.1},
    "acceleration": {"x": 1.8, "y": 0.2, "z": 0.0},
    "rotation": {"yaw": 1.23, "pitch": -0.02, "roll": 0.0}
  }]
}
```

`acceleration` ici correspond aux forces G du paquet source (lateral → x, longitudinal → y,
vertical → z), pas à une dérivée de la vélocité.

## Car Telemetry (packet_id = 6)

Structure par voiture (60 bytes) :

| Offset | Champ | Type | Plage/Unité |
|---|---|---|---|
| 0-1 | speed | uint16 | km/h |
| 2-5 | throttle | float | 0.0-1.0 |
| 6-9 | steer | float | -1.0 (gauche) à 1.0 (droite) |
| 10-13 | brake | float | 0.0-1.0 |
| 14 | clutch | uint8 | 0-100 |
| 15 | gear | int8 | -1=marche arrière, 0=neutre, 1-8 |
| 16-17 | engine_rpm | uint16 | tours/min |
| 18 | drs | uint8 | 0 ou 1 |
| 19 | rev_lights_percent | uint8 | 0-100 |
| 20-21 | rev_lights_bit_value | uint16 | bitmask LED |
| 22-29 | brakes_temperature [4] | uint16 × 4 | °C, par roue |
| 30-33 | tyres_surface_temperature [4] | uint8 × 4 | °C |
| 34-37 | tyres_inner_temperature [4] | uint8 × 4 | °C |
| 38-39 | engine_temperature | uint16 | °C |
| 40-55 | tyres_pressure [4] | float × 4 | PSI |
| 56-59 | surface_type [4] | uint8 × 4 | type de sol par roue |

Après les 22 blocs voiture, le paquet se termine par 3 bytes au niveau paquet :
`mfd_panel_index`, `mfd_panel_index_secondary_player` (uint8) et `suggested_gear` (int8) — non
exploités par ce projet.

### Conversion FastF1 (car_telemetry_packets)

```json
{
  "packet_type": "telemetry",
  "players": [{
    "car_index": 0,
    "speed": 280,
    "throttle": 0.92,
    "brake": 0.0,
    "steering": -0.15,
    "clutch": 0,
    "gear": 5,
    "engine_rpm": 11200,
    "drs": true,
    "temperature": {
      "brake": [380, 380, 380, 380],
      "tire_surface": [92, 92, 92, 92],
      "tire_inner": [104, 104, 104, 104],
      "engine": 94
    },
    "pressure": {"tire": [1.82, 1.82, 1.82, 1.82]}
  }]
}
```

## Lap Data (packet_id = 2)

Structure par voiture (57 bytes). Point important : les temps de secteur ne sont **pas** un seul
champ en millisecondes — ils sont encodés en deux parties (minutes entières + millisecondes) pour
rester compacts tout en supportant des temps > 65 secondes.

| Offset | Champ | Type | Description |
|---|---|---|---|
| 0-3 | last_lap_time_in_ms | uint32 | Temps du dernier tour complet |
| 4-7 | current_lap_time_in_ms | uint32 | Temps du tour en cours |
| 8-9, 10 | sector1_time (ms_part uint16, minutes_part uint8) | — | Secteur 1 = minutes×60 + ms/1000 |
| 11-12, 13 | sector2_time (ms_part uint16, minutes_part uint8) | — | Secteur 2 = minutes×60 + ms/1000 |
| 14-16 | delta_to_car_in_front (ms_part uint16, minutes_part uint8) | — | Non exploité par ce projet |
| 17-19 | delta_to_race_leader (ms_part uint16, minutes_part uint8) | — | Non exploité par ce projet |
| 20-23 | lap_distance | float | mètres, depuis le départ du tour |
| 24-27 | total_distance | float | mètres, depuis le départ de la session |
| 28-31 | safety_car_delta | float | secondes |
| 32 | car_position | uint8 | 1 = leader |
| 33 | current_lap_num | uint8 | — |
| 34-46 | statut course/pénalités (13 × uint8) | — | pit_status, num_pit_stops, sector, current_lap_invalid, penalties, total_warnings, corner_cutting_warnings, num_unserved_drive_through_pens, num_unserved_stop_go_pens, grid_position, driver_status, result_status, pit_lane_timer_active |
| 47-51 | pit_lane_time_in_lane_in_ms (uint16), pit_stop_timer_in_ms (uint16), pit_stop_should_serve_pen (uint8) | — | — |
| 52-56 | speed_trap_fastest_speed (float), speed_trap_fastest_lap (uint8) | — | — |

Après les 22 blocs voiture : `time_trial_pb_car_idx` et `time_trial_rival_car_idx` (uint8 × 2) au
niveau paquet.

### Conversion FastF1 (lap_packets)

```json
{
  "packet_type": "lap",
  "players": [{
    "car_index": 0,
    "lap_time": 119.399,
    "sector_times": [35.812, 41.203],
    "current_lap": 2,
    "position": 1,
    "grid_position": 1,
    "lap_distance": 1204.5,
    "total_distance": 6204.5
  }]
}
```

`lap_time` vient de `last_lap_time_in_ms / 1000`. `sector_times` est reconstruit avec
`minutes_part * 60 + ms_part / 1000` pour chaque secteur.

## Session Data (packet_id = 1)

!!! danger "Périmètre volontairement réduit"
    Le vrai paquet Session du jeu fait plusieurs centaines de bytes (zones de commissaires de
    piste, prévisions météo sur 64 échantillons, dizaines de flags de règles de session...). Ce
    projet ne parse que le **préfixe fixe de 19 bytes** ci-dessous — les champs effectivement
    utilisés par la conversion FastF1 (météo, températures, type de session, circuit). Le reste du
    paquet réel n'est ni parsé ni généré : c'est un choix de portée délibéré, pas un oubli — voir
    les commentaires de `f1_packet_format.py`.

| Offset (après header) | Champ | Type | Description |
|---|---|---|---|
| 0 | weather | uint8 | 0=Clair, 1=Pluie légère, 2=Pluie forte, 3=Orage |
| 1 | track_temperature | int8 | °C |
| 2 | air_temperature | int8 | °C |
| 3 | total_laps | uint8 | Nombre de tours de la session |
| 4-5 | track_length | uint16 | mètres |
| 6 | session_type | uint8 | Essais, qualif, course... |
| 7 | track_id | int8 | Identifiant circuit |
| 8 | formula | uint8 | 0=F1, 1=F2, 2=F3... |
| 9-10 | session_time_left | uint16 | secondes |
| 11-12 | session_duration | uint16 | secondes |
| 13 | pit_speed_limit | uint8 | km/h |
| 14-17 | game_paused, is_spectating, spectator_car_index, sli_pro_native_support | uint8 × 4 | — |
| 18 | num_marshal_zones | uint8 | Toujours 0 ici (zones non générées) |

### Conversion FastF1 (session_packets)

```json
{
  "packet_type": "session",
  "session_uid": "12cca41615ca1203",
  "weather": 0,
  "track_temperature": 32,
  "air_temperature": 24,
  "session_type": 10,
  "track_id": 0
}
```

## Conventions générales

!!! info
    - **Little-endian** partout, sans padding (struct format toujours préfixé `<`)
    - **session_uid** est un uint64 — stocké en MongoDB comme **chaîne hexadécimale** (16
      caractères), car BSON ne supporte que des int64 signés et un uint64 aléatoire peut dépasser
      cette plage
    - **Vecteurs direction** (motion) : int16 normalisés ×32767, pas des floats
    - **Températures pneus/freins** : mélange de uint8 (surface/inner) et uint16 (freins) selon le champ

## Vérification

Chaque taille de bloc ci-dessus est vérifiée par une assertion `struct.calcsize(...) == N` au
chargement de `f1_packet_format.py` — si jamais un format string et sa taille annoncée divergent
(la cause exacte du bug initial de ce projet), le module refuse de s'importer plutôt que de
planter silencieusement à la réception du premier paquet.
