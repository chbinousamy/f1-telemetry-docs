# Wireshark

Capturer et décoder le trafic F1 25 — et comprendre pourquoi Docker le cache parfois.

## Le piège n°1 : le trafic Docker est invisible

!!! danger
    **Wireshark ne capture que ce qui transite par une vraie interface réseau Windows**
    (Ethernet, Wi-Fi, ou le loopback via Npcap). Il ne voit jamais le trafic qui reste entièrement
    à l'intérieur de la couche réseau virtualisée de Docker Desktop — que ce soit entre deux
    conteneurs sur `f1_network`, ou même un aller-retour via `host.docker.internal`. C'est vrai
    **même si les données arrivent bien à destination** (vérifiable via MongoDB) — leur trajet
    emprunte une couche invisible aux outils de capture de l'hôte.

| Scénario | Wireshark voit le trafic ? |
|---|---|
| Process Windows local (générateur ou vrai jeu) → `127.0.0.1:20777` | ✅ Oui — vrai loopback Windows |
| Conteneur → `f1_network` interne → conteneur listener | ❌ Non — réseau bridge Docker interne |
| Conteneur → `host.docker.internal` → conteneur listener | ❌ Non — passerelle virtuelle Docker Desktop |

**Conclusion pratique :** pour observer le trafic dans Wireshark, le process qui *envoie* les
paquets doit être un vrai process Windows — soit le [générateur lancé en local](generator.md)
(`--network local`), soit, plus tard, le vrai jeu F1 25 lui-même (qui est toujours un process
Windows, jamais un conteneur).

## Installation

`https://www.wireshark.org/download/`

L'installateur Windows embarque **Npcap**, qui fournit l'adaptateur *"Adapter for loopback
traffic capture"* nécessaire pour voir le trafic `127.0.0.1`.

## Capturer le trafic du générateur local

1. Lancer le générateur en mode local (envoie vers `127.0.0.1:20777`, le port publié par le conteneur listener) :
   ```bash
   python telemetry_generator.py --network local --replay austrian_2026 --speed-factor 100
   ```
2. Dans Wireshark, sélectionner l'interface **"Adapter for loopback traffic capture"** — pas Ethernet ni Wi-Fi
3. Filtre : `udp.port == 20777`
4. Démarrer la capture (Ctrl+E) — les paquets doivent apparaître immédiatement

## Dissecteur F1 25 pour Wireshark

Un dissecteur Lua dédié, `wireshark_f1_25_dissector.lua` (à la racine du projet), décode le format
binaire exact défini dans `f1_packet_format.py` — au lieu de voir de l'UDP brut en hexadécimal,
Wireshark affiche une couche **F1-25** avec les champs nommés (vitesse, throttle, position
circuit, temps au tour, secteurs...).

### Installation du dissecteur

1. Copier le fichier dans le dossier des plugins personnels de Wireshark :
   ```text
   %APPDATA%\Wireshark\plugins\
   ```
2. Redémarrer Wireshark (ou **Analyze → Reload Lua Plugins**, Ctrl+Shift+L)
3. Filtrer sur `f1_25` pour ne garder que ce protocole, ou combiner avec des champs décodés :
   ```text
   f1_25.telemetry.speed > 300
   f1_25.packet_id == 2
   ```

!!! success "Vérifié"
    Capture `tshark` réelle sur l'interface loopback pendant une simulation locale, dissecteur
    chargé :
    ```text
    127.0.0.1 → 127.0.0.1   F1-25   F1 25 Motion
    127.0.0.1 → 127.0.0.1   F1-25   F1 25 Car Telemetry
    127.0.0.1 → 127.0.0.1   F1-25   F1 25 Lap Data
    127.0.0.1 → 127.0.0.1   F1-25   F1 25 Session
    ```
    275 paquets capturés sans erreur, tous correctement identifiés par type.

## Filtres utiles

| Filtre | Usage |
|---|---|
| `f1_25` | Tout le trafic F1 25 décodé (dissecteur installé) |
| `udp.port == 20777` | Tout le trafic sur le port F1, avec ou sans dissecteur |
| `f1_25.packet_id == 0` | Paquets Motion uniquement |
| `f1_25.packet_id == 6` | Paquets Car Telemetry uniquement |
| `data[6] == 00` | Motion, sans dissecteur (offset fixe du `packet_id`) |
| `f1_25.telemetry.speed > 300` | Pointes de vitesse > 300 km/h |
| `f1_25.lap.car_position == 1` | Paquets lap du leader de la course |

## Vérifier sans ouvrir l'interface graphique

`tshark` (installé avec Wireshark) permet une capture scriptée, pratique pour un test rapide en
ligne de commande :

```bash
"C:\Program Files\Wireshark\tshark.exe" -D
```
Liste les interfaces disponibles — repérer `\Device\NPF_Loopback (Adapter for loopback traffic capture)`.

```bash
"C:\Program Files\Wireshark\tshark.exe" -i "\\Device\\NPF_Loopback" -f "udp port 20777" -a duration:10
```
Capture 10 secondes de trafic sur le loopback. Lancer le générateur local en parallèle dans un
autre terminal pour voir les paquets défiler.

## Dépannage

### Aucun paquet visible

!!! warning
    - **Le plus probable :** le générateur (ou listener) tourne dans un conteneur Docker — voir
      l'avertissement en haut de cette page. Relancer le générateur avec `--network local` en
      process Windows direct.
    - Mauvaise interface sélectionnée — il faut *"Adapter for loopback traffic capture"*, pas
      Ethernet/Wi-Fi, pour du trafic vers `127.0.0.1`
    - Retirer temporairement le filtre pour vérifier qu'il n'y a vraiment aucun trafic du tout,
      avant de soupçonner le filtre lui-même

### Le dissecteur ne s'affiche pas

!!! warning
    - Vérifier le chemin exact : `%APPDATA%\Wireshark\plugins\wireshark_f1_25_dissector.lua`
    - **Help → About Wireshark → Folders** confirme le dossier "Personal Lua Plugins" réellement utilisé
    - **Analyze → Reload Lua Plugins** (Ctrl+Shift+L) après toute modification du script, pas
      besoin de relancer Wireshark entièrement

### Export pour analyse ultérieure

**File → Export Specified Packets → Pcap** — le fichier `.pcapng` garde les paquets décodés et
peut être rouvert plus tard (avec le dissecteur toujours installé pour conserver le décodage).
