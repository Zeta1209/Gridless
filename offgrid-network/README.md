# Station Mesh Hors-Ligne — Guide de Configuration v2

Refonte complète du projet avec :
- **MeshMonitor mis à jour** vers l'architecture multi-sources (v4.x) — l'ancienne config à une seule IP est dépréciée
- **Point d'accès Wi-Fi autonome sur le Pi** : le Heltec et le Pi forment leur propre réseau, sans dépendre du Wi-Fi de la maison ou d'ailleurs
- **Clavier Bluetooth** pour piloter le Raspberry Pi sans clavier filaire
- **Mode Kiosque restreint** : le navigateur ne peut ouvrir QUE MeshMonitor et Wikipedia hors-ligne (Kiwix), rien d'autre — imposé au niveau réseau (proxy filtrant) plutôt que par une politique Chromium qui s'est avérée cassée sur Debian Trixie
- **Wikipedia hors-ligne (Kiwix)**, absent des guides précédents mais nécessaire pour le kiosque
- **Procédure de mise à jour du système** (Partie 9), via un branchement Ethernet temporaire

Ce guide a été testé et corrigé de bout en bout — chaque étape reflète la méthode qui fonctionne réellement, pas juste la théorie. Suivez-le dans l'ordre pour une installation reproductible.

Ce guide remplace entièrement les trois anciens documents (meshtastic.md, meshmonitor.md, offlinemap.md).

---

## Vue d'ensemble de l'architecture

**Le Pi est son propre point d'accès Wi-Fi** — il ne dépend d'aucun réseau externe (maison, chalet, etc.). Le Heltec rejoint ce réseau créé par le Pi, avec une IP fixe réservée. Vous pouvez aussi vous connecter en Bluetooth (BLE) directement au Heltec avec votre cellulaire, en parallèle du Wi-Fi — les deux interfaces fonctionnent indépendamment et simultanément.

```
        ┌───────────────────────────────────────────┐
        │             Raspberry Pi 5                 │
        │        Point d'accès Wi-Fi "MeshStation"   │
        │              IP : 192.168.4.1              │
        │                                             │
        │  ┌─────────────┐ ┌────────────┐            │
        │  │ meshmonitor │ │ tileserver │            │
        │  │   :3001     │ │   :8081    │            │
        │  └─────────────┘ └────────────┘            │
        │         ┌─────────────┐                    │
        │         │   kiwix     │                    │
        │         │   :8082     │                    │
        │         └─────────────┘                    │
        │                                             │
        │  Chromium (kiosque) ── clavier Bluetooth    │
        │  ↳ autorisé UNIQUEMENT :                    │
        │    localhost:3001, localhost:8082           │
        └───────────────┬─────────────────────────────┘
                Wi-Fi "MeshStation" (192.168.4.0/24)
              ┌──────────┴───────────┐
              ▼                       ▼
  ┌────────────────────┐   ┌──────────────────────┐
  │   Heltec LoRa 32    │   │  Votre cellulaire     │
  │  IP fixe: .4.50      │   │ (Wi-Fi MeshStation ou │
  │  (Wi-Fi + BLE)       │   │  BLE direct au Heltec)│
  └────────────────────┘   └──────────────────────┘
```

---

## Partie 1 : Préparation du Raspberry Pi

### 1.1 Flasher Raspberry Pi OS

Avec Raspberry Pi Imager, flashez **Raspberry Pi OS (64-bit)** — prenez la version **avec bureau** (Desktop), pas la Lite, car le mode kiosque a besoin d'un environnement graphique.

Dans les réglages (icône engrenage) :
- Hostname : `meshstation`
- SSH activé (authentification par mot de passe)
- Utilisateur/mot de passe (ex : `mesh` / votre mot de passe)
- Wi-Fi (pour la configuration initiale seulement, peut être désactivé ensuite)
- Fuseau horaire / langue

### 1.2 Premier démarrage et mise à jour

```bash
ssh mesh@meshstation.local
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 1.3 Auto-login (nécessaire pour le kiosque)

```bash
sudo raspi-config
```
Naviguez vers **System Options → Boot / Auto Login → Console Autologin** (⚠️ **pas** "Desktop Autologin"). Le script de kiosque (Partie 7) démarre sa propre session graphique via `startx` depuis une console texte — avec "Desktop Autologin", vous atterrissez directement dans un bureau graphique complet qui ne déclenche jamais ce script, et vous verrez juste le bureau normal au démarrage au lieu du kiosque.

Rebootez après le changement.

### 1.4 Installer Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
exit
```
Reconnectez-vous en SSH puis vérifiez :
```bash
docker --version
docker run hello-world
```

---

## Partie 2 : Jumelage d'un clavier Bluetooth (NOUVEAU)

Ceci remplace le clavier virtuel à l'écran (onboard) utilisé dans l'ancienne version — un vrai clavier Bluetooth est plus fiable et permet une saisie complète (mots de passe, config).

### 2.1 Installer les outils Bluetooth (si absents)

```bash
sudo apt install -y bluetooth bluez bluez-tools
sudo systemctl enable --now bluetooth
```

### 2.2 Mettre le clavier en mode jumelage

Sur la plupart des claviers Bluetooth, maintenez le bouton de jumelage (souvent Fn + un chiffre, ou un bouton dédié) jusqu'à ce que le voyant clignote rapidement.

### 2.3 Jumeler via bluetoothctl

```bash
bluetoothctl
```

Dans l'invite `bluetoothctl` :
```
power on
agent on
default-agent
scan on
```

Attendez que votre clavier apparaisse, ex :
```
[NEW] Device AA:BB:CC:DD:EE:FF Logitech K380
```

Puis :
```
scan off
pair AA:BB:CC:DD:EE:FF
```

Si un code PIN s'affiche à l'écran, tapez-le sur le clavier puis Entrée. Ensuite :
```
trust AA:BB:CC:DD:EE:FF
connect AA:BB:CC:DD:EE:FF
exit
```

### 2.4 Vérifier la connexion persistante

```bash
bluetoothctl info AA:BB:CC:DD:EE:FF
```
Vous devriez voir `Connected: yes` et `Trusted: yes`. Grâce à `trust`, le clavier se reconnectera automatiquement à chaque démarrage du Pi (tant qu'il est allumé et à portée).

### Dépannage Bluetooth

| Symptôme | Solution |
|---|---|
| Clavier non détecté en scan | Remettez-le en mode jumelage, rapprochez-le du Pi |
| Se jumelle mais ne se reconnecte pas après reboot | Vérifiez `trust` : `bluetoothctl trust AA:BB:CC:DD:EE:FF` |
| Frappe saccadée / latence | Désactivez le Wi-Fi 2.4GHz sur le même canal, ou passez le Pi en 5GHz |
| `bluetoothctl` ne trouve pas d'adaptateur | `sudo systemctl status bluetooth`, vérifiez que le Pi 5 a bien le Bluetooth interne activé (`sudo raspi-config` → Interface Options) |

---

## Partie 3 : Configuration du nœud Meshtastic

*(Inchangé dans les grandes lignes — voir aussi la doc officielle : [meshmonitor.org/configuration/serial-bridge.html](https://meshmonitor.org/configuration/serial-bridge.html))*

### 3.1 Flasher le firmware (si nécessaire)

1. Bootloader : maintenez **BOOT**, appuyez sur **RESET**, relâchez **BOOT**
2. Ouvrez https://flasher.meshtastic.org dans **Chrome/Edge**
3. Connectez, sélectionnez **Heltec V3/V4**, version **Latest stable**
4. **Full Erase and Install**, puis **RESET** une fois terminé

### 3.2 Connexion initiale (USB temporaire, une seule fois)

Pour la toute première configuration, branchez le Heltec en USB à **n'importe quel ordinateur** (pas nécessairement le Pi) le temps de configurer le Wi-Fi. Une fois le Wi-Fi actif, vous n'aurez plus jamais besoin du câble.

```bash
ls /dev/ttyACM* /dev/ttyUSB*
lsusb   # cherchez "Espressif USB JTAG/serial debug unit"

pip install meshtastic --break-system-packages
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

meshtastic --port /dev/ttyACM0 --info
```

### 3.3 Créer le point d'accès Wi-Fi autonome sur le Pi (NOUVEAU)

⚠️ **Avertissement important avant de continuer** : le Wi-Fi intégré du Pi est une seule radio — elle ne peut pas être à la fois **cliente** du Wi-Fi de la maison et **point d'accès** en même temps. Dès que vous activez le point d'accès ci-dessous, `wlan0` va lâcher le réseau Wi-Fi actuel et votre session SSH par Wi-Fi sera coupée.

**Pour garder un accès SSH pendant et après cette étape**, branchez un câble Ethernet entre le Pi et votre routeur avant de continuer. Le port Ethernet (`eth0`) reste indépendant du Wi-Fi : vous garderez un accès de gestion depuis la maison, pendant que `wlan0` diffuse `MeshStation` pour le Heltec et le terrain. Trouvez l'IP Ethernet avec :
```bash
ip addr show eth0
```

Si vous êtes déjà déconnecté après avoir lancé les commandes ci-dessous sans Ethernet : connectez votre ordinateur/cellulaire au réseau Wi-Fi `MeshStation` puis faites `ssh mesh@192.168.4.1`.

Pour que le kit fonctionne partout sans dépendre d'un réseau externe, **le Pi devient lui-même le point d'accès Wi-Fi**. Le Heltec (et éventuellement votre cellulaire) s'y connectent, peu importe où vous êtes.

Sur Raspberry Pi OS (Bookworm ou plus récent), NetworkManager gère déjà le Wi-Fi — pas besoin de configurer `hostapd`/`dnsmasq` à la main. **Utilisez la commande `hotspot` dédiée plutôt que de construire le profil manuellement** (`nmcli connection add` + une série de `modify`) : elle configure tout en un coup de façon atomique et est nettement plus fiable — un profil construit à la main peut se retrouver avec des réglages incohérents (ex : mode resté en `infrastructure` au lieu de `ap`) après un reboot.

```bash
sudo nmcli device wifi hotspot ifname wlan0 con-name MeshStation-AP ssid MeshStation password "VotreMotDePasseAP"
```

**Important — forcez la bande 2.4GHz et un canal fixe.** Sans ça, le Pi 5 peut choisir automatiquement le 5GHz pour le point d'accès, une bande que les nœuds Heltec (ESP32) ne voient **pas du tout** — le nœud ne tentera même pas de s'y connecter, sans erreur visible. C'est l'un des pièges les plus faciles à manquer :

```bash
sudo nmcli connection modify MeshStation-AP 802-11-wireless.band bg
sudo nmcli connection modify MeshStation-AP 802-11-wireless.channel 6
sudo nmcli connection modify MeshStation-AP ipv4.addresses 192.168.4.1/24
sudo nmcli connection down MeshStation-AP
sudo nmcli connection up MeshStation-AP
```

Le Pi est maintenant joignable sur `192.168.4.1`, et diffuse un réseau Wi-Fi `MeshStation`.

⚠️ **Étape critique — empêcher `wlan0` de retomber sur le Wi-Fi de la maison.** Par défaut, votre ancien profil Wi-Fi maison (créé par `raspi-config`/netplan lors de l'installation, ex : `netplan-wlan0-Anteverse_2.4GHz`) reste actif avec `autoconnect yes`. NetworkManager peut alors **automatiquement redonner `wlan0` à ce profil** après certains événements système (un `raspi-config`, un reboot, un changement de pays Wi-Fi) — le point d'accès `MeshStation` disparaît alors silencieusement, sans erreur visible, et le Heltec ne peut évidemment plus s'y connecter. C'est un piège facile à ne jamais remarquer puisque le Pi lui-même reste joignable (juste sur un réseau différent).

Désactivez l'auto-connexion de l'ancien profil et donnez une priorité forte au point d'accès :

```bash
nmcli connection show
```
Repérez le nom exact de votre profil Wi-Fi maison dans la liste (souvent `netplan-wlan0-<SSID>` ou juste le nom du SSID), puis :

```bash
sudo nmcli connection modify <nom-du-profil-maison> autoconnect no
sudo nmcli connection modify MeshStation-AP connection.autoconnect yes
sudo nmcli connection modify MeshStation-AP connection.autoconnect-priority 100
sudo nmcli connection up MeshStation-AP
```

**Vérifiez que ça tient après un reboot complet** — c'est le vrai test :
```bash
sudo reboot
```
Puis, après reboot :
```bash
iw dev wlan0 info
```
Vous devez voir `type AP` et `ssid MeshStation`. Si ça montre `type managed` avec un autre SSID, l'ancien profil a repris la main — revérifiez qu'`autoconnect no` a bien été appliqué (`nmcli connection show <nom-du-profil-maison> | grep autoconnect`).

### 3.4 Réserver une IP fixe pour le Heltec sur ce réseau

Pour que le Heltec ait toujours la même IP (nécessaire pour que MeshMonitor le retrouve après chaque redémarrage), on réserve une adresse par son adresse MAC (visible dans le `--info` de la Partie 3.2, champ `macaddr`) :

```bash
sudo mkdir -p /etc/NetworkManager/dnsmasq-shared.d
sudo tee /etc/NetworkManager/dnsmasq-shared.d/heltec.conf > /dev/null << 'EOF'
dhcp-host=8c:fd:49:b7:07:90,192.168.4.50
EOF
sudo systemctl restart NetworkManager
```

Remplacez `8c:fd:49:b7:07:90` par l'adresse MAC réelle de votre Heltec.

### 3.5 Configuration requise du nœud

```bash
set +H   # évite que bash interprète des caractères spéciaux dans les mots de passe

# Région (obligatoire pour transmettre — Canada/USA = US, 915MHz)
meshtastic --port /dev/ttyACM0 --set lora.region US

# Nom de la station de base
meshtastic --port /dev/ttyACM0 --set-owner "Base Station"
meshtastic --port /dev/ttyACM0 --set-owner-short "BASE"

# Si votre nœud a un module GPS : ne configurez PAS de position fixe (voir note ci-dessous).
# Sinon seulement, décommentez et ajustez :
# meshtastic --port /dev/ttyACM0 --set position.fixed_position true
# meshtastic --port /dev/ttyACM0 --setlat 45.5678 --setlon -73.1234 --setalt 150

# Connecter le Heltec au point d'accès Wi-Fi du Pi (PAS au réseau de la maison)
meshtastic --port /dev/ttyACM0 --set network.wifi_enabled true
meshtastic --port /dev/ttyACM0 --set network.wifi_ssid "MeshStation"
meshtastic --port /dev/ttyACM0 --set network.wifi_psk "VotreMotDePasseAP"
```

**Note GPS :** si votre Heltec a un module GPS installé, laissez `position.gps_mode` à `ENABLED` (valeur par défaut) et sautez la position fixe — le nœud déterminera sa position lui-même après quelques minutes à l'extérieur ou près d'une fenêtre. La position fixe n'est utile que pour un nœud **sans** GPS.

**Attention aux mots de passe avec caractères spéciaux** (`!`, `$`, etc.) : le `set +H` ci-dessus évite que bash les interprète comme des raccourcis d'historique ou des variables.

Le nœud va redémarrer et se connecter au réseau `MeshStation`. Le Bluetooth (BLE) reste actif en parallèle — vous pouvez toujours vous y connecter avec l'appli Meshtastic sur votre cellulaire, indépendamment du Pi.

### 3.6 Vérifier la connexion

Après le redémarrage du nœud, **attendez 30 à 60 secondes** avant de tester — le serveur API TCP du Heltec met un moment à démarrer même une fois le Wi-Fi associé, et un test trop hâtif donnera `ECONNREFUSED` alors que tout est en fait normal.

D'abord, confirmez que le Heltec est bien associé au point d'accès (niveau radio) :
```bash
sudo iw dev wlan0 station dump
```
Cherchez une entrée avec l'adresse MAC de votre Heltec (visible dans `--info` → `macaddr`). Si elle n'apparaît pas, le nœud ne s'est pas connecté du tout (revoir la section bande 2.4GHz ci-dessus).

Ensuite, testez la connectivité IP et le port :
```bash
ping 192.168.4.50
nc -zv 192.168.4.50 4403
```

Si ça répond, le Heltec est bien joignable sur son IP fixe — c'est celle-ci qu'on utilise dans le docker-compose à la Partie 4.

### Dépannage Wi-Fi

| Symptôme | Solution |
|---|---|
| Le nœud ne se connecte pas au réseau MeshStation, écran Heltec muet | **Cause la plus fréquente : le point d'accès diffuse en 5GHz.** Vérifiez `nmcli connection show MeshStation-AP \| grep band` — forcez `802-11-wireless.band bg` et un canal (voir 3.3) |
| Le point d'accès refuse de redémarrer après un `raspi-config`/reboot (`ssid-not-found`, mode redevenu `infrastructure`) | Le profil créé à la main peut se corrompre après un reboot. Supprimez-le et recréez-le avec `nmcli device wifi hotspot` (plus robuste) : `sudo nmcli connection delete MeshStation-AP` puis relancez la commande hotspot de la Partie 3.3 |
| Le nœud ne se connecte pas au réseau MeshStation | Vérifiez le SSID/mot de passe : `meshtastic --port /dev/ttyACM0 --get network` |
| L'IP du Heltec n'est pas 192.168.4.50 | Vérifiez que la MAC dans `heltec.conf` correspond bien à celle du nœud (`--info` → `macaddr`) |
| `nmcli` dit que le point d'accès ne démarre pas | Vérifiez que le Wi-Fi du Pi n'est pas bloqué : `rfkill list`, puis `rfkill unblock wifi` |
| SSH coupé juste après avoir activé le point d'accès | Normal — une seule radio Wi-Fi ne peut pas être cliente + AP en même temps. Reconnectez-vous via `ssh mesh@192.168.4.1` en rejoignant le réseau `MeshStation`, ou utilisez l'Ethernet (voir avertissement Partie 3.3) |
| Le point d'accès fonctionnait puis a disparu sans raison, `iw dev wlan0 info` montre `type managed` avec le SSID de la maison | `wlan0` est retombé sur l'ancien profil Wi-Fi maison — vérifiez `autoconnect no` sur ce profil et la priorité sur `MeshStation-AP` (voir la mise en garde à la fin de la Partie 3.3) |
| `ECONNREFUSED` sur le port 4403 juste après un reboot du nœud | Normal — attendez 30-60s de plus, le serveur API démarre après l'association Wi-Fi (voir 3.6) |
| MeshMonitor ne se connecte plus au nœud | `ping 192.168.4.50` puis `nc -zv 192.168.4.50 4403` pour tester le port TCP |
| Je veux redonner le nœud en USB direct à un ordi | Aucun problème — le Wi-Fi et l'USB ne sont pas exclusifs, le nœud répond aux deux |

---

## Partie 4 : Installation de MeshMonitor (v4.x — architecture multi-sources)

⚠️ **Changements importants depuis votre ancienne installation** :
1. MeshMonitor 4.0+ gère plusieurs "Sources" (Meshtastic, MeshCore, MQTT) depuis un seul tableau de bord. La variable `MESHTASTIC_NODE_IP` ne fait plus que **amorcer la première source au premier démarrage** — toute la gestion se fait ensuite depuis **Dashboard → Sources** dans l'interface web, pas dans le docker-compose.
2. Le Heltec étant maintenant en **Wi-Fi sur le point d'accès autonome du Pi** (Partie 3) et non branché en USB, **il n'y a plus de conteneur `serial-bridge`** — MeshMonitor se connecte directement à l'IP fixe du nœud sur ce réseau, exactement comme n'importe quel appareil.
3. **MeshMonitor doit tourner en `network_mode: host`, pas en réseau bridge classique.** Le réseau bridge par défaut de Docker (avec un mappage de port `8080:3001`) passe par le NAT du conteneur, qui ne route pas correctement vers le sous-réseau du point d'accès Wi-Fi (`192.168.4.0/24`) créé par NetworkManager — le Pi lui-même arrive à joindre le Heltec, mais le conteneur se fait bloquer (`ECONNREFUSED` malgré un port qui répond bien depuis l'hôte). En mode `host`, le conteneur partage directement la pile réseau du Pi et le problème disparaît. La contrepartie : MeshMonitor est alors accessible sur le port natif `3001` (pas `8080`), puisqu'il n'y a plus de mappage de port en mode host.

### 4.1 Créer le projet

```bash
mkdir -p ~/meshmonitor/tiles ~/meshmonitor/kiwix
cd ~/meshmonitor
```

### 4.2 docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
services:
  meshmonitor:
    image: ghcr.io/yeraze/meshmonitor:latest
    container_name: meshmonitor
    network_mode: host
    volumes:
      - meshmonitor-data:/data
    environment:
      # Amorce uniquement la 1ère source au premier démarrage.
      # Ensuite : Dashboard → Sources pour tout gérer (renommage, sources additionnelles, etc.)
      # IP fixe réservée pour le Heltec sur le point d'accès du Pi (Partie 3.4)
      - MESHTASTIC_NODE_IP=192.168.4.50
      # Port 3001 (pas 8080) car network_mode: host n'utilise pas de mappage de port
      - ALLOWED_ORIGINS=http://localhost:3001,http://192.168.4.1:3001,http://meshstation.local:3001
      - TZ=America/Toronto
    restart: unless-stopped

  tileserver:
    image: maptiler/tileserver-gl-light:latest
    container_name: tileserver
    ports:
      - "8081:8080"
    volumes:
      - ./tiles:/data
    restart: unless-stopped

  kiwix:
    image: ghcr.io/kiwix/kiwix-serve:latest
    container_name: kiwix
    command: ["*.zim"]
    ports:
      - "8082:8080"
    volumes:
      - ./kiwix:/data
    restart: unless-stopped

volumes:
  meshmonitor-data:
EOF
```

**Notes :**
- `192.168.4.50` est l'IP fixe réservée pour le Heltec sur le point d'accès du Pi (Partie 3.4) — pas besoin de la changer si vous avez suivi le guide
- MeshMonitor est accessible sur **le port 3001** (pas 8080) à cause du `network_mode: host` — c'est le port natif de l'application
- `tileserver` et `kiwix` restent en réseau bridge classique avec mappage de port (`8081`, `8082`) — ils n'ont pas besoin de joindre le sous-réseau Wi-Fi du Heltec, donc pas de problème de NAT pour eux
- `ALLOWED_ORIGINS` doit inclure toutes les façons dont vous accédez à l'interface (localhost, IP du point d'accès, hostname)
- **Choix de tag d'image** : `:latest` (stable, ~hebdomadaire) est recommandé. `:4.13` verrouille une ligne mineure, `:4.13.0` verrouille une version exacte et n'évoluera jamais.

### 4.3 Démarrer

```bash
docker compose up -d
docker compose ps
```

### 4.4 Premier login et gestion des sources

1. Ouvrez `http://meshstation.local:3001` (ou `http://192.168.4.1:3001`)
2. Connectez-vous : `admin` / `changeme` → **changez le mot de passe immédiatement** (icône profil → Change Password)
3. Allez dans **Dashboard → Sources** : vous devriez voir la source amorcée automatiquement depuis `MESHTASTIC_NODE_IP`. C'est ici que vous ajouterez d'éventuels autres nœuds plus tard (pas dans le docker-compose).

### 4.5 Position fixe de la station de base (si pas de GPS)

Dans MeshMonitor : onglet **Device** → **Position Settings** → activez **Fixed Position** → entrez lat/lon/altitude → **Save**.

### 4.6 Note : `kiwix` et `tileserver` en boucle de redémarrage, c'est normal pour l'instant

Si `docker compose ps` montre `kiwix` et/ou `tileserver` en `Restarting` juste après le premier `docker compose up -d`, ne vous inquiétez pas — ils quittent proprement (exit code 0) simplement parce qu'**aucun fichier de données n'est encore présent** dans `~/meshmonitor/tiles/` ou `~/meshmonitor/kiwix/`. MeshMonitor, lui, continue de tourner normalement pendant ce temps. Une fois les fichiers `.mbtiles`/`.zim` en place (Parties 5 et 6), ils se stabilisent.

---

## Partie 5 : Cartes hors-ligne (TileServer GL Light)

TileServer GL Light gère maintenant **à la fois les tuiles vectorielles (.pbf) et raster (.png)** — plus besoin de choisir une image "Full" avec des dépendances fragiles (`sharp`).

⚠️ **Limite importante : un seul `.mbtiles` à la fois en mode auto-détection.** Sans fichier `config.json`, TileServer GL Light ne scanne fiablement qu'**un seul** fichier `.mbtiles` présent dans `~/meshmonitor/tiles/` — s'il y en a plusieurs, seul l'un d'eux (pas nécessairement celui voulu) sera servi, et les autres seront silencieusement ignorés. Deux options :
- **Simple** : ne gardez qu'un seul fichier `.mbtiles` dans le dossier (ex : votre région complète)
- **Plusieurs régions/tilesets** : créez un `config.json` explicite listant chaque fichier (voir 5.4)

### 5.1 Télécharger des tuiles

- Vectorielles (recommandé, plus léger) : [MapTiler OSM](https://www.maptiler.com/on-prem-datasets/)
- Raster : [OpenMapTiles](https://openmaptiles.org/downloads/)

Placez le(s) fichier(s) `.mbtiles` dans `~/meshmonitor/tiles/`.

### 5.2 Vérifier le rendu

```
http://192.168.4.1:8081
```

### 5.3 Ajouter la tuile dans MeshMonitor

**Settings → Map Settings → Custom Tile Servers → + Add Custom Tile Server**

⚠️ **Utilisez `192.168.4.1` (pas `localhost`) dans l'URL** — c'est votre navigateur qui va chercher les tuiles directement (pas le conteneur MeshMonitor côté serveur), donc il faut une adresse que le navigateur peut réellement joindre. `localhost` ne fonctionne que si vous accédez à l'interface depuis le Pi lui-même (ex : mode kiosque).

⚠️ **Le `v3` dans les URLs ci-dessous n'est PAS un numéro de version** — c'est simplement le nom de la source telle que nommée dans un `config.json` d'exemple de la documentation officielle. En mode auto-détection (sans `config.json`), le nom réel de la source dépend de l'implémentation et peut différer. Si l'URL avec `v3` ne fonctionne pas, vérifiez le nom exact via l'interface `http://192.168.4.1:8081` (section **Data**) ou utilisez un `config.json` explicite (5.4) où **vous choisissez vous-même** ce nom.

Pour des tuiles **vectorielles** :
```
Name: Local Vector Tiles
URL: http://192.168.4.1:8081/data/v3/{z}/{x}/{y}.pbf
Attribution: © OpenStreetMap contributors
Max Zoom: 14
```

Pour des tuiles **raster** :
```
Name: Local OSM
URL: http://192.168.4.1:8081/styles/osm-bright/{z}/{x}/{y}.png
Attribution: © OpenStreetMap contributors
Max Zoom: 18
```

Cliquez **Save**.

**Sélectionner le tileset actif :** le choix est maintenant séparé pour les modes clair et sombre — un **Light Mode Tileset** et un **Dark Mode Tileset** distincts, chacun appliqué automatiquement selon le thème actif. Toujours dans **Map Settings**, sélectionnez votre tileset custom dans les deux menus déroulants si vous voulez qu'il s'applique peu importe le thème.

### 5.4 (Optionnel) Servir plusieurs fichiers .mbtiles avec un config.json explicite

Si vous voulez plusieurs régions/tilesets en même temps, créez un fichier `config.json` dans `~/meshmonitor/tiles/` qui liste chaque fichier explicitement — c'est la méthode fiable, contrairement à l'auto-détection :

```bash
cat > ~/meshmonitor/tiles/config.json << 'EOF'
{
  "options": {
    "paths": {
      "root": "",
      "mbtiles": ""
    }
  },
  "data": {
    "canada": {
      "mbtiles": "osm-2020-02-10-v3.11_north-america_canada.mbtiles"
    },
    "autre-region": {
      "mbtiles": "nom-du-deuxieme-fichier.mbtiles"
    }
  }
}
EOF

cd ~/meshmonitor
docker compose restart tileserver
```

L'URL dans MeshMonitor utilise alors le nom **que vous avez choisi** comme clé (`canada`, `autre-region`), pas `v3` :
```
http://192.168.4.1:8081/data/canada/{z}/{x}/{y}.pbf
```

---

## Partie 6 : Wikipedia hors-ligne avec Kiwix (NOUVEAU)

Kiwix sert des archives Wikipedia (fichiers `.zim`) totalement hors-ligne.

### 6.1 Télécharger un fichier ZIM

Parcourez le catalogue et choisissez une version selon l'espace disponible :
```
https://library.kiwix.org/
```

Exemples (Wikipedia en français, sans images, très compact) :
```bash
cd ~/meshmonitor/kiwix
wget https://download.kiwix.org/zim/wikipedia/wikipedia_fr_all_nopic.zim
```

**Astuce espace disque :** la version `_all_nopic` (texte seul) fait quelques Go ; `_all_maxi` (avec images) peut dépasser 100 Go. Choisissez selon votre carte SD/SSD.

### 6.2 Le conteneur Kiwix est déjà dans le docker-compose de la Partie 4

Il sert automatiquement tous les fichiers `.zim` présents dans `~/meshmonitor/kiwix/`. Redémarrez pour qu'il les détecte :
```bash
cd ~/meshmonitor
docker compose restart kiwix
```

### 6.3 Vérifier

```
http://192.168.4.1:8082
```

Vous devriez voir la page d'accueil Kiwix avec votre archive Wikipedia listée.

---

## Partie 7 : Mode Kiosque restreint (MeshMonitor + Wikipedia UNIQUEMENT)

C'est le cœur de la demande : un écran qui **ne peut afficher que MeshMonitor et Wikipedia hors-ligne**, sans barre d'adresse, sans possibilité de naviguer ailleurs.

### 7.1 Installer les composants graphiques minimaux

```bash
sudo apt install -y --no-install-recommends \
  xserver-xorg xserver-xorg-legacy x11-xserver-utils xinit openbox \
  chromium unclutter
```

⚠️ **Nom du paquet variable selon la version de Raspberry Pi OS** : sur les versions récentes, le navigateur s'appelle `chromium` (pas `chromium-browser`). Vérifiez lequel est réellement installé avant de continuer :
```bash
command -v chromium chromium-browser
```
Utilisez le nom qui répond dans toutes les commandes ci-dessous — ce guide utilise `chromium`, ajustez si nécessaire.

### 7.1bis Autoriser X à démarrer sans les droits root (Raspberry Pi OS récent)

Sur les versions récentes de Raspberry Pi OS (Debian Trixie+), les utilisateurs normaux n'ont plus le droit de démarrer le serveur X par défaut — il faut l'autoriser explicitement, sinon vous aurez un écran noir ou une erreur `Cannot open /dev/tty0 (Permission denied)` :

```bash
sudo nano /etc/X11/Xwrapper.config
```

Assurez-vous que le fichier contient :
```
allowed_users=anybody
needs_root_rights=yes
```

Puis ajoutez votre utilisateur aux bons groupes :
```bash
sudo usermod -aG tty,video $USER
```

**Important : testez `startx` uniquement depuis l'écran physique du Pi, jamais par SSH** — SSH n'a pas accès à un vrai terminal (`/dev/tty0`), donc un test par SSH échouera toujours avec une erreur qui n'a rien à voir avec un vrai problème.

### 7.1ter Installer les polices (icônes/emojis)

Sans ça, les icônes 📡📖 de la page de lancement (Partie 7.2) ne s'affichent pas :

```bash
sudo apt install -y fontconfig fonts-dejavu fonts-noto-color-emoji
sudo fc-cache -f
```

### 7.2 Créer la page de lancement locale

Cette page sert de "page d'accueil" du kiosque, avec deux gros boutons.

```bash
mkdir -p ~/kiosk
cat > ~/kiosk/launcher.html << 'EOF'
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Station Mesh</title>
<style>
  body { margin:0; height:100vh; display:flex; align-items:center; justify-content:center;
         background:#1e1e2e; font-family: sans-serif; gap:40px; }
  a { display:flex; flex-direction:column; align-items:center; justify-content:center;
      width:320px; height:320px; border-radius:24px; text-decoration:none;
      color:white; font-size:28px; font-weight:bold; }
  .mesh { background:#89b4fa; }
  .wiki { background:#a6e3a1; }
</style>
</head>
<body>
  <a class="mesh" href="http://localhost:3001">📡<br>MeshMonitor</a>
  <a class="wiki" href="http://localhost:8082">📖<br>Wikipedia</a>
</body>
</html>
EOF
```

⚠️ **Ne pas ouvrir ce fichier directement en `file://` dans Chromium** — le filtrage d'URL de la Partie 7.3 gère mal ce schéma. Le script de la Partie 7.4 sert cette page via un petit serveur HTTP local (`http://localhost:8090`) à la place, ce qui évite complètement le problème.

### 7.3 Restreindre le navigateur au niveau réseau (proxy filtrant)

⚠️ **Pourquoi pas la politique Chromium `URLBlocklist`/`URLAllowlist` ?** C'est l'approche officiellement documentée par Google/Chromium, mais un **bug confirmé du paquet `chromium` de Debian Trixie** (introduit mi-2025, toujours présent) fait que ces politiques s'affichent bien dans `chrome://policy` mais **ne sont pas réellement appliquées** — le navigateur bloque de façon incohérente, y compris ses propres URLs pourtant listées dans l'allowlist. Plutôt que de dépendre d'un mécanisme cassé côté navigateur, on impose la restriction **au niveau du système** avec un proxy filtrant local (`tinyproxy`) : Chromium n'a alors plus aucun rôle à jouer dans le blocage, c'est Linux qui l'impose, indépendamment de tout bug de Chromium.

#### Installer et configurer tinyproxy

```bash
sudo apt install -y tinyproxy
```

Créer la liste de filtrage (schéma complet inclus, requis avec `FilterURLs`) :
```bash
sudo tee /etc/tinyproxy/filter > /dev/null << 'EOF'
^http://localhost:3001
^http://localhost:8082
^http://localhost:8090
^http://127\.0\.0\.1:3001
^http://127\.0\.0\.1:8082
^http://127\.0\.0\.1:8090
EOF
```

Éditer la config principale :
```bash
sudo nano /etc/tinyproxy/tinyproxy.conf
```

Assurez-vous que ces lignes sont présentes (ajoutez-les si absentes, décommentez si présentes en commentaire) :
```
Port 8888
Listen 127.0.0.1
Allow 127.0.0.1
Filter "/etc/tinyproxy/filter"
FilterDefaultDeny Yes
FilterURLs Yes
```

⚠️ **`FilterURLs Yes` est indispensable** — sans cette ligne, tinyproxy compare vos règles seulement contre le **nom d'hôte** (`localhost`), jamais le port ni le chemin, et vos règles ne correspondront jamais à rien.

Redémarrer et activer au boot :
```bash
sudo systemctl restart tinyproxy
sudo systemctl enable tinyproxy
```

#### Valider le filtrage avant de continuer

```bash
curl -x http://127.0.0.1:8888 http://localhost:3001
curl -x http://127.0.0.1:8888 http://localhost:8082
curl -x http://127.0.0.1:8888 http://localhost:8090/launcher.html
curl -x http://127.0.0.1:8888 http://example.com
```
Les trois premiers doivent retourner du contenu normal ; le dernier (`example.com`) doit retourner une page **"403 Filtered"**. Ne continuez pas tant que ce test n'est pas concluant — ça évite de chasser un problème de navigateur qui serait en fait un problème de proxy.

### 7.4 Script de démarrage du kiosque

Le script inclut : un serveur HTTP local pour la page de lancement, le proxy filtrant, et une **journalisation dans `~/kiosk.log`** — indispensable pour diagnostiquer si Chromium ne se lance pas ou plante silencieusement (écran noir).

```bash
cat > ~/kiosk.sh << 'EOF'
#!/bin/bash

exec > ~/kiosk.log 2>&1
echo "=== Kiosk starting: $(date) ==="

xset s off
xset s noblank
xset -dpms

unclutter -idle 0.5 -root &

# Servir la page de lancement en HTTP local (évite les soucis de filtrage file://)
(cd ~/kiosk && python3 -m http.server 8090 --bind 127.0.0.1 &)

# Attendre que les services Docker et le mini-serveur soient prêts
sleep 15

echo "Launching browser..."

chromium \
  --kiosk \
  --proxy-server="127.0.0.1:8888" \
  --proxy-bypass-list="<-loopback>" \
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --disable-pinch \
  --overscroll-history-navigation=0 \
  --check-for-update-interval=604800 \
  http://localhost:8090/launcher.html

echo "Browser exited with code $?: $(date)"
EOF

chmod +x ~/kiosk.sh
```

⚠️ Si votre système utilise `chromium-browser` plutôt que `chromium` (vérifié à l'étape 7.1), remplacez le nom dans ce script en conséquence. Le flag `--proxy-bypass-list="<-loopback>"` est requis — sans lui, Chromium contourne automatiquement le proxy pour `localhost`, ce qui rendrait le filtrage inutile.

**En cas de problème**, consultez le log après un reboot :
```bash
cat ~/kiosk.log
```
Une ligne `command not found` indique un mauvais nom de paquet ; un code de sortie non nul juste après le lancement indique un plantage de Chromium — le message d'erreur juste au-dessus donne généralement la cause exacte.

### 7.5 Démarrage automatique au boot

```bash
echo "exec openbox-session" > ~/.xinitrc

mkdir -p ~/.config/openbox
echo "~/kiosk.sh &" > ~/.config/openbox/autostart

echo '[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && startx' >> ~/.bash_profile
```

Comme l'auto-login desktop est déjà activé (Partie 1.3), le kiosque se lance automatiquement à chaque démarrage.

```bash
sudo reboot
```

### 7.6 Revenir au bureau normal (maintenance) / naviguer entre les deux apps

**Pour revenir à la page de lancement depuis MeshMonitor ou Kiwix** : même en mode `--kiosk` (barre d'adresse cachée), le raccourci **Alt + Flèche gauche** (retour en arrière dans l'historique) fonctionne avec le clavier Bluetooth.

Le clavier Bluetooth étant fonctionnel, vous pouvez aussi fermer Chromium complètement avec **Alt+F4**, ou vous connecter en SSH depuis un autre appareil pour faire de la maintenance sans toucher à l'écran (`pkill chromium`, en ajustant le nom si votre paquet s'appelle `chromium-browser`).

Pour désactiver temporairement le kiosque :
```bash
mv ~/.config/openbox/autostart ~/.config/openbox/autostart.disabled
sudo reboot
```
Pour le réactiver :
```bash
mv ~/.config/openbox/autostart.disabled ~/.config/openbox/autostart
sudo reboot
```

### Dépannage Kiosque

| Symptôme | Solution |
|---|---|
| Le Pi démarre juste sur le bureau normal, pas le kiosque | Vous êtes probablement en **Desktop Autologin** au lieu de **Console Autologin** — voir Partie 1.3. C'est la cause la plus fréquente |
| `Cannot open /dev/tty0 (Permission denied)` en testant `startx` | Soit vous testez par **SSH** (invalide, testez sur l'écran physique seulement), soit `Xwrapper.config` n'autorise pas les utilisateurs normaux à lancer X — voir Partie 7.1bis |
| Écran noir après reboot, X démarre puis se termine proprement quelques secondes après | Openbox ou Chromium plante silencieusement. Consultez `~/kiosk.log` pour voir l'erreur exacte |
| `chromium-browser: command not found` dans `~/kiosk.log` | Le paquet s'appelle `chromium` sur votre système, pas `chromium-browser`. Corrigez le nom dans `~/kiosk.sh` (voir 7.1 et 7.4) |
| Icônes 📡📖 invisibles sur la page de lancement | Polices manquantes — voir Partie 7.1ter (`fonts-noto-color-emoji`) |
| Page "Filtered" (tinyproxy) sur MeshMonitor/Kiwix/la page de lancement | Testez d'abord en dehors du navigateur : `curl -x http://127.0.0.1:8888 http://localhost:3001`. Si ça échoue aussi, vérifiez `FilterURLs Yes` dans `tinyproxy.conf` et que vos règles dans `/etc/tinyproxy/filter` incluent bien le schéma `http://` complet (voir 7.3) |
| Toujours "Filtered" même après avoir corrigé `tinyproxy.conf`/`filter` | Assurez-vous d'avoir bien retiré l'ancienne politique Chromium si vous l'aviez testée : `sudo rm -f /etc/chromium/policies/managed/policies.json`, puis rebootez — les deux mécanismes peuvent se chevaucher |
| Un site externe passe quand même à travers le proxy | Vérifiez `--proxy-bypass-list="<-loopback>"` dans `kiosk.sh` — sans ce flag exact, Chromium peut contourner le proxy pour certaines requêtes locales |
| Le kiosque se lance sur un écran noir (mais pas de crash dans le log) | Augmentez le `sleep 15` dans kiosk.sh — Docker n'a peut-être pas fini de démarrer |
| Impossible de fermer Chromium | Utilisez SSH depuis un autre appareil : `ssh mesh@meshstation.local` puis `pkill chromium` (ou `chromium-browser` selon votre paquet) |
| Le clavier Bluetooth ne répond pas au démarrage | Vérifiez `bluetoothctl info <MAC>` — le `trust` doit être actif (voir Partie 2.4) |

---

## Partie 8 : Maintenance

### Commandes Docker courantes

```bash
cd ~/meshmonitor

# Logs
docker compose logs -f
docker compose logs -f meshmonitor
docker compose logs -f kiwix

# Redémarrer
docker compose restart

# Mettre à jour (MeshMonitor a aussi un bouton "Update" intégré dans l'interface)
docker compose pull
docker compose up -d

# Arrêter
docker compose down
```

### Sauvegarde

```bash
docker run --rm \
  -v meshmonitor_meshmonitor-data:/data \
  -v ~/backups:/backup \
  alpine tar czf /backup/meshmonitor-backup-$(date +%Y%m%d).tar.gz /data
```

MeshMonitor 4.x inclut aussi une fonction **Backup & Restore** complète directement dans l'interface (Settings → System).

---

## Partie 9 : Mettre à jour le système (NOUVEAU)

Comme le Wi-Fi (`wlan0`) est entièrement dédié au point d'accès `MeshStation`, le Pi n'a plus d'accès Internet par ce biais — il faut **brancher un câble Ethernet** temporairement pour mettre à jour le système, les conteneurs Docker, et vérifier le firmware du Heltec.

### 9.1 Brancher l'Ethernet

Branchez un câble entre le port Ethernet du Pi et votre routeur (avec accès Internet). Confirmez l'obtention d'une IP :

```bash
ip addr show eth0
```

Le point d'accès `MeshStation` sur `wlan0` continue de fonctionner normalement pendant ce temps — les deux interfaces sont indépendantes, brancher l'Ethernet ne coupe rien pour le Heltec.

### 9.2 Mettre à jour le système d'exploitation

```bash
sudo apt update && sudo apt upgrade -y
```

Si le noyau ou des composants critiques ont été mis à jour, redémarrez :
```bash
sudo reboot
```

⚠️ **Après ce reboot, vérifiez que le point d'accès a bien repris** (voir l'avertissement de la Partie 3.3 sur `wlan0` qui peut retomber sur l'ancien profil Wi-Fi maison) :
```bash
iw dev wlan0 info
```
Confirmez `type AP` et `ssid MeshStation`.

### 9.3 Mettre à jour les conteneurs Docker (MeshMonitor, TileServer, Kiwix)

```bash
cd ~/meshmonitor
docker compose pull
docker compose up -d
docker compose ps
```

MeshMonitor a aussi un bouton **Update** intégré dans son interface web (Settings → System) qui peut suffire pour lui seul, sans passer par la ligne de commande.

### 9.4 Vérifier le firmware du Heltec (optionnel)

Branchez temporairement le Heltec en USB (à n'importe quel ordinateur avec accès Internet, pas nécessairement le Pi) et comparez votre version actuelle à la dernière disponible :

```bash
meshtastic --port /dev/ttyACM0 --info
```
Notez `firmwareVersion`, puis comparez avec la dernière version stable sur https://flasher.meshtastic.org — suivez la Partie 3.1 pour reflasher si une mise à jour est souhaitée. **Ce n'est pas obligatoire à chaque fois** ; ne le faites que si vous rencontrez un bug spécifique ou voulez une nouvelle fonctionnalité.

### 9.5 Débrancher l'Ethernet

Une fois les mises à jour terminées, débranchez le câble — le kit reprend son fonctionnement entièrement autonome sur `wlan0`/`MeshStation`.

### Fréquence recommandée

Il n'y a pas d'urgence à mettre à jour souvent un système qui fonctionne bien hors-ligne — une vérification tous les 1-3 mois (ou avant un déploiement important sur le terrain) est largement suffisante.

---

## Référence rapide

### URLs des services

| Service | URL | Port |
|---|---|---|
| MeshMonitor | http://192.168.4.1:3001 (ou meshstation.local:3001) | 3001 *(network_mode: host, pas de mappage)* |
| TileServer | http://192.168.4.1:8081 | 8081 |
| Kiwix (Wikipedia) | http://192.168.4.1:8082 | 8082 |
| Point d'accès Wi-Fi du Pi | SSID `MeshStation`, IP 192.168.4.1 | — |
| Heltec (Wi-Fi, TCP Meshtastic) | 192.168.4.50 *(IP fixe réservée)* | 4403 |
| SSH | meshstation.local ou 192.168.4.1 | 22 |

### Identifiants par défaut

| Service | Utilisateur | Mot de passe |
|---|---|---|
| MeshMonitor | admin | changeme *(à changer immédiatement)* |
| Pi SSH | mesh | *(le vôtre)* |

### Emplacements des fichiers

| Élément | Emplacement |
|---|---|
| Docker Compose | `~/meshmonitor/docker-compose.yml` |
| Tuiles de carte | `~/meshmonitor/tiles/` |
| Fichiers ZIM (Wikipedia) | `~/meshmonitor/kiwix/` |
| Page de lancement kiosque | `~/kiosk/launcher.html` (servie via `http://localhost:8090`) |
| Filtre proxy (restriction réseau) | `/etc/tinyproxy/filter` |
| Config proxy | `/etc/tinyproxy/tinyproxy.conf` |
| Script de démarrage kiosque | `~/kiosk.sh` |
| Journal de débogage du kiosque | `~/kiosk.log` |

---

## Prochaines étapes

1. Configurer des nœuds portables additionnels (T-Echo, T-Deck) — ils apparaîtront automatiquement via **Dashboard → Sources**
2. Tester la portée et la couverture sur le terrain
3. Ajouter d'autres fichiers `.zim` dans Kiwix (dictionnaires, Wiktionnaire, guides de survie, etc.) — ils apparaîtront automatiquement sur la page d'accueil Kiwix
4. Ajuster la portée du point d'accès Wi-Fi du Pi si besoin (canal, puissance) selon l'environnement de déploiement
