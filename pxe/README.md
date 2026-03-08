# PXE — Ubuntu 24.04 Autoinstall avec LUKS + LVM

## Fichiers

| Fichier | Rôle |
|---------|------|
| `user-data` | Config autoinstall complète (stockage, réseau, SSH, paquets) |
| `meta-data` | Métadonnées cloud-init (hostname, instance-id) |

---

## Ce que fait l'autoinstall

```
/dev/sda
├─sda1   512M   fat32     /boot/efi   ← EFI (GRUB)
├─sda2     2G   ext4      /boot       ← kernel + initramfs (non chiffré)
└─sda3   reste  LUKS                  ← chiffré
  └─luks0        LVM PV
    └─ubuntu--vg-ubuntu--lv  ext4  /  ← racine
```

Paquets pré-installés : `dropbear-initramfs`, `clevis-luks`, `clevis-initramfs`, `clevis-systemd`

Dropbear configuré dans l'initramfs via `late-commands` → déverrouillage SSH possible dès le premier boot.

---

## À adapter avant utilisation

Dans `user-data`, modifier **obligatoirement** :

| Champ | Défaut | Description |
|-------|--------|-------------|
| `path: /dev/sda` | `/dev/sda` | Disque cible |
| `ens3` | `ens3` | Interface réseau |
| `192.168.1.10/24` | — | IP statique du serveur |
| `gateway4` | `192.168.1.1` | Passerelle |
| `hostname` | `srv-ubuntu` | Nom du serveur |
| `password` | — | Hash SHA-512 (`openssl passwd -6`) |
| `authorized-keys` | — | Clé SSH admin |
| `key: "MotDePasseTemporaire..."` | — | Mot de passe LUKS fallback |
| Clé Dropbear (`late-commands`) | — | Clé SSH pour déverrouillage initramfs |

**Générer le hash du mot de passe :**
```bash
echo 'MonMotDePasse' | openssl passwd -6 -stdin
```

---

## Servir les fichiers via HTTP (serveur PXE)

Les fichiers `user-data` et `meta-data` doivent être accessibles via HTTP depuis le serveur PXE :

```bash
# Depuis le dossier pxe/
python3 -m http.server 3003
# → http://IP_PXE:3003/user-data
# → http://IP_PXE:3003/meta-data
```

---

## Paramètre GRUB/UEFI PXE à ajouter à la ligne de boot Ubuntu

```
autoinstall ds=nocloud;s=http://IP_PXE:3003/
```

---

## Workflow complet après installation

```
1. PXE boot → autoinstall → LUKS + LVM créés, Dropbear configuré
                                    ↓
2. Premier boot → Dropbear disponible sur port 2222
   ssh -p 2222 -i ~/.ssh/id_ed25519 root@IP → cryptroot-unlock
                                    ↓
3. ansible-playbook tang-server.yml      (sur le serveur Tang)
4. ansible-playbook clevis-client.yml    (sur le serveur client)
   → clevis bind LUKS → Tang
                                    ↓
5. Reboot suivant → déchiffrement automatique ✅
   (Dropbear reste en fallback si Tang injoignable)
```
