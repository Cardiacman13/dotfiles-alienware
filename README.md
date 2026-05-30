# Notes & fichiers de conf — Alienware (Linux)

## 1) Commande Steam

**Steam → Propriétés du jeu → Options de lancement** :

```bash
__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 PROTON_DLSS_UPGRADE=1 DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE_RENDER_PRESET_SELECTION=render_preset_l PROTON_ENABLE_WAYLAND=1 game-performance %command%
```

En cas de craquements audio ou stutters je peux soit utiliser que mes P-Cores : `WINE_CPU_TOPOLOGY=16:0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15` soit utiliser un scheduler qui gère mieux ça comme flash en monde low latency.

**Notes rapides :**

* `WINE_CPU_TOPOLOGY=16:...` : Ne concerne que les CPU Intel E-cores + P-Cores et encore plus précisément mon 14900hx, utilise les 16 premiers threads de mon CPU pour le jeu ignorant les E-Cores
* `DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE_RENDER_PRESET_SELECTION=render_preset_l` Important de forcer le modèle l, update les dlls du dlss n'est pas suffisant


Améliorer la vitesse de téléchargement et compilation des shaders sur mes 32 threads:

```bash
echo "@nClientDownloadEnableHTTP2PlatformLinux 0" >> ~/.steam/steam/steam_dev.cfg
echo "@fDownloadRateImprovementToAddAnotherConnection 1.0" >> ~/.steam/steam/steam_dev.cfg
echo "unShaderBackgroundProcessingThreads 32" >> ~/.steam/steam/steam_dev.cfg
```
---

## 2) Augmenter la taille du shader cache NVIDIA (12 Go)

On crée un fichier d’environnement pour l’utilisateur 

### Création du fichier

```bash
mkdir -p ~/.config/environment.d
nano ~/.config/environment.d/gaming.conf
```

### Contenu à mettre dedans

```ini
# NVIDIA shader disk cache
__GL_SHADER_DISK_CACHE_SIZE=12000000000
```

---

## 3) WirePlumber : empêcher l’auto-switch en mode “Headset” (HSP/HFP)

Objectif : éviter que le Bluetooth bascule tout seul en profil micro-casque (qualité audio dégradée).

### Créer le fichier de conf

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
nano ~/.config/wireplumber/wireplumber.conf.d/51-disable-hsp-autoswitch.conf
```

### Contenu

```ini
wireplumber.settings = {
  bluetooth.autoswitch-to-headset-profile = false
}

monitor.bluez.properties = {
  bluez5.roles = [ a2dp_sink a2dp_source ]
}
```

### Redémarrer WirePlumber

```bash
systemctl --user restart wireplumber
```
Voici **la section RAID0 Btrfs minimale** à ajouter telle quelle à tes notes.
Je l’ai écrite pour être **courte, actionnable et sans blabla**, dans le même style que le reste.

---

## 4) RAID0 logiciel Btrfs (NVMe) — méthode minimale

Objectif : utiliser **2 SSD NVMe** comme **un seul volume** avec **performances maximales**, sans RAID firmware (Intel RST).

### Principes retenus

* **DATA** → `RAID0`
* **METADATA / SYSTEM** → `RAID1` (sécurité minimale du FS)
* Boot EFI **hors RAID** (normal)

---

### Étape 1 — Préparer le 2ᵉ SSD (⚠️ efface le disque)

à adapter (`nvme0n1` ici).

```bash
sudo wipefs -a -f /dev/nvme0n1
sudo parted /dev/nvme0n1 --script mklabel gpt
sudo parted /dev/nvme0n1 --script mkpart primary 1MiB 100%
sudo partprobe
```

---

### Étape 2 — Ajouter le disque au Btrfs existant (`/`)

```bash
sudo btrfs device add -f /dev/nvme0n1p1 /
sudo btrfs filesystem show /
```

---

### Étape 3 — Conversion en RAID0 (DATA) + RAID1 (METADATA)

```bash
sudo btrfs balance start -dconvert=raid0 -mconvert=raid1 /
sudo btrfs balance status /
```

---

### Étape 4 — Vérification

```bash
sudo btrfs filesystem df /
sudo btrfs filesystem usage -T /
sudo btrfs device stats /
```

Attendu :

* `Data, RAID0`
* `Metadata, RAID1`
* erreurs = `0`

## 5) Unbloat / fastboot

`sudo nano /boot/limine.conf timeout: 0` puis `sudo limine-update`

`sudo nano /etc/mkinitcpio.conf` virer base et plymouth et mettre `COMPRESSION="lz4"` et `COMPRESSION_OPTIONS=("--fast=65537")` puis `sudo mkinitcpio -P`

`sudo pacman -Rns plymouth plymouth-kcm cachyos-plymouth-theme cachyos-plymouth-bootanimation`

`systemctl mask NetworkManager-wait-online.service cachyos-rate-mirrors.timer`

`sudo pacman -S --asexplicit linux-firmware-intel linux-firmware-whence linux-firmware-nvidia inux-firmware-realtek`
`sudo pacman -Rns linux-firmware linux-firmware-amdgpu linux-firmware-radeon linux-firmware-atheros linux-firmware-broadcom linux-firmware-mediatek linux-firmware-cirrus linux-firmware-other`
