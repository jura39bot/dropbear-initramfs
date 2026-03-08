# Kickstart — RHEL 10

Installation automatique RHEL 10 avec **EFI + /boot ext4 + LUKS + LVM + ext4**.

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
| `sshkey` | — | Clé SSH publique |
| `--drives=sda` | `sda` | Disque cible |
| `--passphrase=` | `MotDePasseTemporaire123!` | Mot de passe LUKS fallback |

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
