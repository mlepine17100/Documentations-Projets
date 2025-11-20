# Documentation : Renommer un Volume Group (VG) LVM sur Debian 13

Cette documentation décrit la procédure technique pour changer le nom d'un Groupe de Volumes (Volume Group - VG) sous Debian.

> [!WARNING]
> **Avertissement Critique**
> * **Sauvegardez vos données** avant toute manipulation sur les partitions.
> * Cette procédure est décrite pour un VG de données. Si vous renommez le VG contenant la racine (`/`), voir la section **"Cas Particulier"** en bas de document.

---

## 1. Identification

Avant de commencer, listez les VGs pour récupérer le nom exact actuel.

```bash
sudo vgs
```

*Sortie exemple :*
```text
  VG        #PV #LV #SN Attr   VSize    VFree
  vg_ancien   1   2   0 wz--n- 500.00g  10.00g
```

---

## 2. Renommage du Volume Group

La commande `vgrename` prend en paramètre l'ancien nom puis le nouveau nom.

**Syntaxe :**
```bash
sudo vgrename <ancien_nom> <nouveau_nom>
```

**Exemple :**
```bash
sudo vgrename vg_ancien vg_data
```

*Si succès :*
> `Volume group "vg_ancien" successfully renamed to "vg_data"`

---

## 3. Mise à jour de la configuration système

Une fois le VG renommé, le système ne pourra plus monter les partitions automatiquement si les fichiers de configuration ne sont pas mis à jour.

### A. Modifier `/etc/fstab`

C'est le fichier qui gère les points de montage au démarrage.

1.  Ouvrez le fichier :
    ```bash
    sudo nano /etc/fstab
    ```
2.  Cherchez les lignes contenant l'ancien nom (ex: `/dev/mapper/vg_ancien-...`).
3.  Remplacez par le nouveau nom (ex: `/dev/mapper/vg_data-...`).
4.  Sauvegardez (`Ctrl+O`) et quittez (`Ctrl+X`).

### B. Modifier `/etc/initramfs-tools/conf.d/resume` (Optionnel)

*Uniquement si votre partition SWAP se trouve sur ce VG.*

1.  Vérifiez le contenu :
    ```bash
    cat /etc/initramfs-tools/conf.d/resume
    ```
2.  Si le fichier mentionne l'ancien VG, éditez-le pour mettre le nouveau chemin :
    ```bash
    RESUME=/dev/mapper/vg_data-lv_swap
    ```

---

## 4. Prise en compte au démarrage (Initramfs & Grub)

Même si le VG ne contient pas le système racine, il est recommandé de mettre à jour l'environnement de démarrage pour éviter des délais d'attente ou des erreurs lors du boot.

**1. Mettre à jour l'image Initramfs :**
```bash
sudo update-initramfs -u -k all
```

**2. Mettre à jour le chargeur Grub :**
```bash
sudo update-grub
```

---

## 5. Vérification

Vérifiez que le changement est bien pris en compte par le noyau LVM :

```bash
sudo lvs
```

Si tout est correct, un redémarrage est conseillé pour valider le montage automatique :

```bash
sudo systemctl reboot
```

---

## 🚨 Cas Particulier : Renommer le VG Racine (Root)

Si le VG à renommer contient le système d'exploitation (`/`) :

1.  **NE PAS** effectuer la procédure depuis le système en cours d'exécution.
2.  Démarrez sur une **Live USB** (Debian Live ou SystemRescue).
3.  Ouvrez un terminal et faites le `vgrename`.
4.  Montez le système en mode **chroot** (`/mnt`).
5.  Effectuez les modifications (`fstab`, `grub`) et les mises à jour (`update-initramfs`, `update-grub`) à l'intérieur du chroot.