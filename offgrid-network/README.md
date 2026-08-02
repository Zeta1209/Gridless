# Station Mesh Hors-Ligne — Guide de Configuration v2
 
Refonte complète du projet avec :
- **MeshMonitor mis à jour** vers l'architecture multi-sources (v4.x) — l'ancienne config à une seule IP est dépréciée
- **Point d'accès Wi-Fi autonome sur le Pi** : le Heltec et le Pi forment leur propre réseau, sans dépendre du Wi-Fi de la maison ou d'ailleurs
- **Clavier Bluetooth** pour piloter le Raspberry Pi sans clavier filaire
- **Mode Kiosque restreint** : le navigateur ne peut ouvrir QUE MeshMonitor et Wikipedia hors-ligne (Kiwix), rien d'autre
- **Wikipedia hors-ligne (Kiwix)**, absent des guides précédents mais nécessaire pour le kiosque
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
        │  │   :8080     │ │   :8081    │            │
        │  └─────────────┘ └────────────┘            │
        │         ┌─────────────┐                    │
        │         │   kiwix     │                    │
        │         │   :8082     │                    │
        │         └─────────────┘                    │
        │                                             │
        │  Chromium (kiosque) ── clavier Bluetooth    │
        │  ↳ autorisé UNIQUEMENT :                    │
        │    localhost:8080, localhost:8082           │
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
Naviguez vers **System Options → Boot / Auto Login → Desktop Autologin**, puis rebootez.
 
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
 
Sur Raspberry Pi OS (Bookworm ou plus récent), NetworkManager gère déjà le Wi-Fi — pas besoin de configurer `hostapd`/`dnsmasq` à la main :
 
```bash
sudo nmcli connection add type wifi ifname wlan0 con-name MeshStation-AP autoconnect yes ssid MeshStation
sudo nmcli connection modify MeshStation-AP 802-11-wireless.mode ap 802-11-wireless.band bg
sudo nmcli connection modify MeshStation-AP wifi-sec.key-mgmt wpa-psk
sudo nmcli connection modify MeshStation-AP wifi-sec.psk "VotreMotDePasseAP"
sudo nmcli connection modify MeshStation-AP ipv4.method shared
sudo nmcli connection modify MeshStation-AP ipv4.addresses 192.168.4.1/24
sudo nmcli connection up MeshStation-AP
```
 
Le Pi est maintenant joignable sur `192.168.4.1`, et diffuse un réseau Wi-Fi `MeshStation`.
 
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
 
```bash
ping 192.168.4.50
telnet 192.168.4.50 4403
```
 
Si ça répond, le Heltec est bien joignable sur son IP fixe — c'est celle-ci qu'on utilise dans le docker-compose à la Partie 4.
 
### Dépannage Wi-Fi
 
| Symptôme | Solution |
|---|---|
| Le nœud ne se connecte pas au réseau MeshStation | Vérifiez le SSID/mot de passe : `meshtastic --port /dev/ttyACM0 --get network` |
| L'IP du Heltec n'est pas 192.168.4.50 | Vérifiez que la MAC dans `heltec.conf` correspond bien à celle du nœud (`--info` → `macaddr`) |
| `nmcli` dit que le point d'accès ne démarre pas | Vérifiez que le Wi-Fi du Pi n'est pas bloqué : `rfkill list`, puis `rfkill unblock wifi` |
| SSH coupé juste après avoir activé le point d'accès | Normal — une seule radio Wi-Fi ne peut pas être cliente + AP en même temps. Reconnectez-vous via `ssh mesh@192.168.4.1` en rejoignant le réseau `MeshStation`, ou utilisez l'Ethernet (voir avertissement Partie 3.3) |
| MeshMonitor ne se connecte plus au nœud | `ping 192.168.4.50` puis `telnet 192.168.4.50 4403` pour tester le port TCP |
| Je veux redonner le nœud en USB direct à un ordi | Aucun problème — le Wi-Fi et l'USB ne sont pas exclusifs, le nœud répond aux deux |
 
---
 
## Partie 4 : Installation de MeshMonitor (v4.x — architecture multi-sources)
 
⚠️ **Changements importants depuis votre ancienne installation** :
1. MeshMonitor 4.0+ gère plusieurs "Sources" (Meshtastic, MeshCore, MQTT) depuis un seul tableau de bord. La variable `MESHTASTIC_NODE_IP` ne fait plus que **amorcer la première source au premier démarrage** — toute la gestion se fait ensuite depuis **Dashboard → Sources** dans l'interface web, pas dans le docker-compose.
2. Le Heltec étant maintenant en **Wi-Fi sur le point d'accès autonome du Pi** (Partie 3) et non branché en USB, **il n'y a plus de conteneur `serial-bridge`** — MeshMonitor se connecte directement à l'IP fixe du nœud sur ce réseau, exactement comme n'importe quel appareil.
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
    ports:
      - "8080:3001"
    volumes:
      - meshmonitor-data:/data
    environment:
      # Amorce uniquement la 1ère source au premier démarrage.
      # Ensuite : Dashboard → Sources pour tout gérer (renommage, sources additionnelles, etc.)
      # IP fixe réservée pour le Heltec sur le point d'accès du Pi (Partie 3.4)
      - MESHTASTIC_NODE_IP=192.168.4.50
      - ALLOWED_ORIGINS=http://localhost:8080,http://192.168.4.1:8080,http://meshstation.local:8080
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
- `ALLOWED_ORIGINS` doit inclure toutes les façons dont vous accédez à l'interface (localhost, IP du point d'accès, hostname)
- **Choix de tag d'image** : `:latest` (stable, ~hebdomadaire) est recommandé. `:4.13` verrouille une ligne mineure, `:4.13.0` verrouille une version exacte et n'évoluera jamais.
### 4.3 Démarrer
 
```bash
docker compose up -d
docker compose ps
```
 
### 4.4 Premier login et gestion des sources
 
1. Ouvrez `http://meshstation.local:8080`
2. Connectez-vous : `admin` / `changeme` → **changez le mot de passe immédiatement** (icône profil → Change Password)
3. Allez dans **Dashboard → Sources** : vous devriez voir la source amorcée automatiquement depuis `MESHTASTIC_NODE_IP`. C'est ici que vous ajouterez d'éventuels autres nœuds plus tard (pas dans le docker-compose).
### 4.5 Position fixe de la station de base (si pas de GPS)
 
Dans MeshMonitor : onglet **Device** → **Position Settings** → activez **Fixed Position** → entrez lat/lon/altitude → **Save**.
 
---
 
## Partie 5 : Cartes hors-ligne (TileServer GL Light)
 
TileServer GL Light gère maintenant **à la fois les tuiles vectorielles (.pbf) et raster (.png)** — plus besoin de choisir une image "Full" avec des dépendances fragiles (`sharp`).
 
### 5.1 Télécharger des tuiles
 
- Vectorielles (recommandé, plus léger) : [MapTiler OSM](https://www.maptiler.com/on-prem-datasets/)
- Raster : [OpenMapTiles](https://openmaptiles.org/downloads/)
Placez le(s) fichier(s) `.mbtiles` dans `~/meshmonitor/tiles/`.
 
### 5.2 Vérifier le rendu
 
```
http://meshstation.local:8081
```
 
### 5.3 Ajouter la tuile dans MeshMonitor
 
**Settings → Map Settings → Custom Tile Servers → + Add Custom Tile Server**
 
Pour des tuiles **vectorielles** :
```
Name: Local Vector Tiles
URL: http://meshstation.local:8081/data/v3/{z}/{x}/{y}.pbf
Attribution: © OpenStreetMap contributors
Max Zoom: 14
```
 
Pour des tuiles **raster** :
```
Name: Local OSM
URL: http://meshstation.local:8081/styles/osm-bright/{z}/{x}/{y}.png
Attribution: © OpenStreetMap contributors
Max Zoom: 18
```
 
Sauvegardez, puis sélectionnez le tileset dans le menu déroulant **Map Tileset**.
 
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
http://meshstation.local:8082
```
 
Vous devriez voir la page d'accueil Kiwix avec votre archive Wikipedia listée.
 
---
 
## Partie 7 : Mode Kiosque restreint (MeshMonitor + Wikipedia UNIQUEMENT)
 
C'est le cœur de la demande : un écran qui **ne peut afficher que MeshMonitor et Wikipedia hors-ligne**, sans barre d'adresse, sans possibilité de naviguer ailleurs.
 
### 7.1 Installer les composants graphiques minimaux
 
```bash
sudo apt install -y --no-install-recommends \
  xserver-xorg x11-xserver-utils xinit openbox \
  chromium-browser unclutter
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
  <a class="mesh" href="http://localhost:8080">📡<br>MeshMonitor</a>
  <a class="wiki" href="http://localhost:8082">📖<br>Wikipedia</a>
</body>
</html>
EOF
```
 
### 7.3 Restreindre Chromium via une politique gérée (URLAllowlist)
 
C'est ce qui empêche réellement de sortir des deux applications autorisées, même via un raccourci clavier ou un lien externe.
 
```bash
sudo mkdir -p /etc/chromium/policies/managed
sudo tee /etc/chromium/policies/managed/policies.json > /dev/null << 'EOF'
{
  "URLBlocklist": ["*"],
  "URLAllowlist": [
    "http://localhost:8080/*",
    "http://localhost:8082/*",
    "file:///home/mesh/kiosk/*"
  ],
  "IncognitoModeAvailability": 1,
  "DeveloperToolsAvailability": 2,
  "BrowserAddPersonEnabled": false,
  "BrowserGuestModeEnabled": false,
  "TranslateEnabled": false,
  "DefaultBrowserSettingEnabled": false
}
EOF
```
 
**Explication :**
- `URLBlocklist: ["*"]` bloque absolument tout par défaut
- `URLAllowlist` réautorise seulement MeshMonitor, Kiwix et la page de lancement locale (l'allowlist a priorité sur le blocklist)
- `DeveloperToolsAvailability: 2` empêche l'ouverture des outils de développement (échappatoire classique)
- `IncognitoModeAvailability: 1` désactive le mode navigation privée
Adaptez `/home/mesh/kiosk/` si votre nom d'utilisateur diffère de `mesh`.
 
### 7.4 Script de démarrage du kiosque
 
```bash
cat > ~/kiosk.sh << 'EOF'
#!/bin/bash
 
xset s off
xset s noblank
xset -dpms
 
unclutter -idle 0.5 -root &
 
# Attendre que les services Docker soient prêts
sleep 15
 
chromium-browser \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --disable-pinch \
  --overscroll-history-navigation=0 \
  --check-for-update-interval=604800 \
  file:///home/mesh/kiosk/launcher.html
EOF
 
chmod +x ~/kiosk.sh
```
 
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
 
### 7.6 Revenir au bureau normal (maintenance)
 
Le clavier Bluetooth étant fonctionnel, vous pouvez fermer Chromium avec **Alt+F4**, ou vous connecter en SSH depuis un autre appareil pour faire de la maintenance sans toucher à l'écran.
 
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
| Page "bloquée par votre administrateur" sur MeshMonitor/Kiwix | Vérifiez que le port dans l'URL correspond exactement à `URLAllowlist` (8080/8082) |
| Le kiosque se lance sur un écran noir | Augmentez le `sleep 15` dans kiosk.sh — Docker n'a peut-être pas fini de démarrer |
| Impossible de fermer Chromium | Utilisez SSH depuis un autre appareil : `ssh mesh@meshstation.local` puis `pkill chromium-browser` |
| Le clavier Bluetooth ne répond pas au démarrage | Vérifiez `bluetoothctl info <MAC>` — le `trust` doit être actif (voir Partie 2.4) |
| La politique JSON n'est pas appliquée | Redémarrez Chromium complètement (`sudo reboot`) ; vérifiez la syntaxe JSON avec `python3 -m json.tool /etc/chromium/policies/managed/policies.json` |
 
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
 
## Référence rapide
 
### URLs des services
 
| Service | URL | Port |
|---|---|---|
| MeshMonitor | http://192.168.4.1:8080 (ou meshstation.local:8080) | 8080 |
| TileServer | http://192.168.4.1:8081 | 8081 |
| Kiwix (Wikipedia) | http://192.168.4.1:8082 | 8082 |
| Point d'accès Wi-Fi du Pi | SSID `MeshStation`, IP 192.168.4.1 | — |
| Heltec (Wi-Fi, TCP Meshtastic) | 192.168.4.50 *(IP fixe réservée)* | 4403 |
| SSH | meshstation.local | 22 |
 
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
| Page de lancement kiosque | `~/kiosk/launcher.html` |
| Politique Chromium | `/etc/chromium/policies/managed/policies.json` |
| Script de démarrage kiosque | `~/kiosk.sh` |
 
---
 
## Prochaines étapes
 
1. Configurer des nœuds portables additionnels (T-Echo, T-Deck) — ils apparaîtront automatiquement via **Dashboard → Sources**
2. Tester la portée et la couverture sur le terrain
3. Ajouter d'autres fichiers `.zim` dans Kiwix (dictionnaires, Wiktionnaire, guides de survie, etc.) — ils apparaîtront automatiquement sur la page d'accueil Kiwix
4. Ajuster la portée du point d'accès Wi-Fi du Pi si besoin (canal, puissance) selon l'environnement de déploiement
