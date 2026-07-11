# Playbook Ansible - Setup NAS Fedora (kernel-longterm + ZFS + Podman)

Point de départ, à faire évoluer avec le reste de la stack (Jellyfin, Sonarr, Radarr,
qBittorrent, VoidAuth, Kodi, gamescope, Moonlight).

## Prérequis

```bash
ansible-galaxy collection install community.general
```

Adapte `inventory.ini` avec l'IP réelle du serveur, et `group_vars/all.yml` avec :
- la version de kernel-longterm à utiliser (vérifie sur le forum Fedora Discussion
  quelle branche est recommandée pour ta version de Fedora avant de lancer)
- le nom exact de ton pool ZFS
- les chemins de montage SSD/HDD si tu les changes

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
- Le rôle `podman` s'arrête après la config du stockage — il manque le déploiement
  du compose (Jellyfin/Sonarr/Radarr/qBittorrent/VoidAuth), à ajouter dans un rôle
  séparé une fois cette base validée.
- Rien n'est encore prévu pour Kodi/gamescope/Moonlight (service systemd,
  script dispatcher) — ça fera un rôle `htpc` à part, sur le vrai matériel
  uniquement (pas testable en VM comme on l'a vu).
- Avant chaque exécution qui touche au kernel, vérifie manuellement la
  compatibilité ZFS avec la nouvelle version sur
  https://github.com/openzfs/zfs/releases



Ajouter droits SELinux
Ajouter droits SSD
Ajouter droits HDD