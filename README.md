# Playbook Ansible - Setup NAS Fedora (kernel-longterm + ZFS + Podman)

Point de départ, à faire évoluer avec le reste de la stack (Jellyfin, Sonarr, Radarr,
qBittorrent, VoidAuth, Kodi, gamescope, Moonlight).

## Prérequis

```bash
ansible-galaxy collection install community.general ansible.posix
```

Adapte `inventory.ini` avec l'IP réelle du serveur, et `group_vars/all.yml` avec :
- la version de kernel-longterm à utiliser (vérifie sur le forum Fedora Discussion
  quelle branche est recommandée pour ta version de Fedora avant de lancer)
- le nom exact de ton pool ZFS
- les chemins de montage SSD/HDD si tu les changes

### Secrets (VoidAuth)

```bash
cp group_vars/vault.yml.example group_vars/vault.yml
$EDITOR group_vars/vault.yml   # renseigne vault_db_password et vault_voidauth_storage_key
ansible-vault encrypt group_vars/vault.yml
```

Lance ensuite le playbook avec `--ask-vault-pass` (ou `--vault-password-file`).
Ne jamais committer `group_vars/vault.yml` en clair.

## Lancer le playbook

```bash
ansible-playbook -i inventory.ini site.yml
```

Le rôle `kernel_longterm` nécessite un reboot en plein milieu du processus.
**Relance simplement `ansible-playbook` une seconde fois** après le premier passage :
le playbook détecte automatiquement qu'il tourne déjà sur le kernel-longterm et
enchaîne sur la suppression du kernel Fedora standard, puis sur les rôles ZFS et Podman.

## Lancer seulement une partie (tags)

```bash
ansible-playbook -i inventory.ini site.yml --tags zfs
ansible-playbook -i inventory.ini site.yml --tags podman
```

## Limites actuelles / à faire ensuite

- Le rôle `zfs` importe un pool **existant** (`zpool import`), il ne crée pas de
  pool RAID-Z1 from scratch — à ajouter si tu veux aussi couvrir ce cas (ex. pour
  la VM de test avec des disques vierges).
- Jellyfin/Sonarr/Radarr/Prowlarr/Seerr/qBittorrent/VoidAuth sont maintenant
  déployés via quadlets Podman (rootless, un utilisateur système par groupe de
  services). Le Caddyfile expose désormais tous ces services par nom de domaine :
  - `jellyfin.{{ public_domain }}` et `seerr.{{ public_domain }}` passent par
    VoidAuth (`forward_auth`) — pense à créer au moins une invitation depuis
    `sso.{{ public_domain }}` avant de t'en servir, sinon impossible de se logger.
  - `sonarr.{{ public_domain }}`, `radarr.{{ public_domain }}`,
    `prowlarr.{{ public_domain }}` et `qbittorrent.{{ public_domain }}` passent
    par les DEUX couches : filtre LAN (`remote_ip private_ranges` + `abort`,
    coupe tout ce qui n'est pas dans une plage privée) ET VoidAuth
    (`forward_auth`, même compte que Jellyfin/Seerr). Pense à passer le
    réglage "Authentication Required" de chacune de ces apps sur `Enabled`
    dans leur interface (pas géré par ce playbook) — ne compte pas sur leur
    mode "Disabled for Local Addresses", qui a eu une vraie CVE
    d'authentication bypass (CVE-2026-30975 sur Sonarr) quand le reverse
    proxy devant n'est pas irréprochable sur X-Forwarded-For. VoidAuth
    devient la seule porte d'entrée fiable pour ces quatre apps.
  - Ça suppose que `{{ public_domain }}` (et ses sous-domaines) résout vers
    l'IP LAN du NAS pour tes appareils locaux — via `/etc/hosts`, un DNS local
    (Pi-hole, AdGuard, la box...) ou du split-DNS. Rien ne le fait automatiquement.
- Rien n'est encore prévu pour Kodi/gamescope/Moonlight (service systemd,
  script dispatcher) — ça fera un rôle `htpc` à part, sur le vrai matériel
  uniquement (pas testable en VM comme on l'a vu).
- Avant chaque exécution qui touche au kernel, vérifie manuellement la
  compatibilité ZFS avec la nouvelle version sur
  https://github.com/openzfs/zfs/releases



Ajouter droits SELinux
Ajouter droits SSD
Ajouter droits HDD