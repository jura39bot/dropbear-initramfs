# Kickstart — RHEL 10

Installation automatique RHEL 10 avec **EFI + /boot ext4 + LUKS + LVM + ext4**.

---

## SSH dans l'initramfs — dracut-sshd

Sur RHEL, **Dropbear n'est pas dans les repos officiels**. Le kickstart utilise à la place **`dracut-sshd`** (EPEL) qui intègre OpenSSH dans l'initramfs dracut.

```
Ubuntu 24.04  → dropbear-initramfs  (port 2222)
RHEL 10       → dracut-sshd         (port 22, via EPEL)
```

Connexion au boot pour déverrouillage manuel :
```bash
ssh -i ~/.ssh/id_ed25519 root@192.168.1.10
# puis :
systemd-tty-ask-password-agent --query
# ou :
echo "MotDePasse" > /run/cryptsetup-keys.d/...
```

---

## À adapter dans `ks.cfg`

| Champ | Défaut | Description |
|-------|--------|-------------|
| `url --url=` | — | Miroir RHEL HTTP |
| `--ip=` | `192.168.1.10` | IP statique du serveur |
| `--gateway=` | `192.168.1.1` | Passerelle |
| `--device=` | `ens3` | Interface réseau |
| `--hostname=` | `srv-rhel` | Nom du serveur |
| `rootpw` | — | Hash SHA-512 (`openssl passwd -6`) |
| `user --password=` | — | Hash SHA-512 admin |
| `sshkey` | — | Clé SSH publique (accès normal) |
| `authorized_keys` dans `%post` | — | Clé SSH pour déverrouillage initramfs |
| `--drives=sda` | `sda` | Disque cible |
| `--passphrase=` | `MotDePasseTemporaire123!` | Mot de passe LUKS fallback |
| `kernel_cmdline ip=` | `192.168.1.10` | IP réseau dans l'initramfs |

**Générer un hash de mot de passe :**
```bash
echo 'MonMotDePasse' | openssl passwd -6 -stdin
```

---

## Servir le kickstart via HTTP (PXE)

```bash
cd kickstart/
python3 -m http.server 3004
# → http://IP_PXE:3004/ks.cfg
```

---

## Paramètre de boot GRUB/UEFI à ajouter

```
inst.ks=http://IP_PXE:3004/ks.cfg
```

---

## Structure disque après installation

```
/dev/sda
├─sda1   512M   fat32     /boot/efi   ← EFI
├─sda2     2G   ext4      /boot       ← non chiffré
└─sda3   reste  LUKS
  └─rhel--vg-rhel--lv    ext4   /     ← LVM + ext4
```

---

## Workflow complet après installation

```
1. PXE boot → kickstart → LUKS + LVM créés
                              ↓
2. ansible-playbook tang-server-rhel.yml   (serveur Tang)
3. ansible-playbook clevis-client-rhel.yml (client — détecte LUKS auto)
                              ↓
4. Reboot → déchiffrement automatique ✅
```
