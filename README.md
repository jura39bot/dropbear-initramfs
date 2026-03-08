# dropbear-initramfs

Playbook Ansible pour installer et configurer **Dropbear dans initramfs** sur **Ubuntu 24.04 Server**.

Permet le déverrouillage SSH de LUKS au boot, **sans mot de passe** (clé SSH uniquement).

---

## Ce que fait le playbook

1. Lit le fichier `/etc/netplan/*.yaml` sur la cible pour extraire automatiquement :
   - Adresse IP statique
   - Netmask
   - Gateway
   - Nom de l'interface (ethernet ou bridge)
2. Installe `dropbear-initramfs`
3. Configure le réseau dans `/etc/initramfs-tools/initramfs.conf`
4. Ajoute la clé SSH publique dans `/etc/dropbear/initramfs/authorized_keys`
5. Configure Dropbear (port 2222, **auth mot de passe désactivée**)
6. Régénère l'initramfs
7. Vérifie que Dropbear est bien inclus

---

## Prérequis

- Ubuntu 24.04 Server
- LUKS activé à l'installation
- IP statique configurée dans netplan
- `netaddr` Python installé sur le contrôleur Ansible :
  ```bash
  pip install netaddr
  ```

---

## Usage standalone

```bash
git clone https://github.com/jura39bot/dropbear-initramfs.git
cd dropbear-initramfs

# Adapter l'inventaire
vi inventory/hosts.yml

# Lancer le playbook
ansible-playbook playbook.yml \
  -e "ssh_public_key='ssh-ed25519 AAAA... user@host'"
```

---

## Usage AWX / Ansible Tower

1. Créer un **Project** pointant sur ce repo
2. Créer un **Job Template** avec le playbook `playbook.yml`
3. Ajouter une **Survey** avec la variable :

| Nom de variable   | Type  | Obligatoire | Description              |
|-------------------|-------|-------------|--------------------------|
| `ssh_public_key`  | Texte | ✅ Oui       | Clé publique SSH (contenu complet) |

4. Lancer le job — le playbook détecte automatiquement la config réseau depuis netplan.

---

## Variables

| Variable | Défaut | Description |
|----------|--------|-------------|
| `ssh_public_key` | — | **Requise** (survey AWX) — clé publique SSH |
| `dropbear_port` | `2222` | Port d'écoute Dropbear |
| `dropbear_options` | `-p 2222 -s -j -k -I 60` | Options Dropbear complètes |

---

## Connexion au boot

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 root@<IP_SERVEUR>
# puis :
cryptroot-unlock
```

---

## Structure

```
.
├── ansible.cfg
├── playbook.yml          # Dropbear dans initramfs
├── clevis-client.yml     # Clevis client (déchiffrement LUKS via Tang)
├── tang-server.yml       # Serveur Tang
├── inventory/
│   └── hosts.yml         # Inventaire exemple
├── pxe/
│   ├── user-data         # Autoinstall Ubuntu 24.04 (LUKS + LVM + EFI)
│   ├── meta-data         # Métadonnées cloud-init
│   └── README.md         # Guide PXE
└── README.md
```

---

## Workflow complet (PXE → Tang → Clevis)

```
1. pxe/user-data  → Installation Ubuntu 24.04 avec LUKS + LVM
                     Dropbear pré-configuré dans initramfs
                              ↓
2. tang-server.yml → Serveur Tang opérationnel
                              ↓
3. clevis-client.yml → LUKS bindé à Tang, initramfs régénéré
                              ↓
4. Reboot → déchiffrement automatique ✅
            Dropbear en fallback si Tang injoignable
```

---

## Tang + Clevis — déchiffrement LUKS automatique au boot

### 1. Installer le serveur Tang

```bash
ansible-playbook -i inventory/hosts.yml tang-server.yml
# → affiche l'URL et le thumbprint à utiliser côté client
```

### 2. Configurer les clients (serveurs à déchiffrer)

```bash
ansible-playbook -i inventory/hosts.yml clevis-client.yml \
  -e "tang_url=http://192.168.1.20" \
  -e "tang_thumbprint=<THUMBPRINT_AFFICHÉ>"
```

**Variables survey AWX :**

| Variable | Type | Obligatoire | Description |
|----------|------|-------------|-------------|
| `tang_url` | Texte | ✅ | URL du serveur Tang ex: `http://192.168.1.20` |
| `tang_thumbprint` | Texte | ❌ | Empreinte SHA1 — si vide, acceptée automatiquement |

Le playbook détecte **automatiquement** toutes les partitions `crypto_LUKS` du système.
