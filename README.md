# Notes & fichiers de conf — Alienware (Linux)

## 1) Commande Steam

**Steam → Propriétés du jeu → Options de lancement** :

```bash
WINE_CPU_TOPOLOGY=16:0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15 __GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1 PROTON_DLSS_UPGRADE=1 DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE_RENDER_PRESET_SELECTION=render_preset_m PROTON_ENABLE_WAYLAND=1 PROTON_USE_NTSYNC=1 game-performance %command%
```

**Notes rapides :**

* `WINE_CPU_TOPOLOGY=16:...` : Ne concerne que les CPU Intel E-cores + P-Cores, utilise les 16 premiers threads de mon CPU pour le jeu ignorant les E-Cores
* `DXVK_NVAPI_DRS_NGX_DLSS_SR_OVERRIDE_RENDER_PRESET_SELECTION=render_preset_m` Important de forcer le modèle m update les dlls du dlss n'est pas suffisant

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

Adapter le disque (`nvme0n1` ici).

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

---

### Maintenance minimale recommandée

```bash
# Scrub mensuel
sudo systemctl enable --now btrfs-scrub@-.timer

# TRIM hebdomadaire (SSD)
sudo systemctl enable --now fstrim.timer
```

---
