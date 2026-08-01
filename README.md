# Playbook Ansible - Setup NAS Fedora (kernel-longterm + ZFS + Podman)

Point de départ, à faire évoluer avec le reste de la stack (Jellyfin, Sonarr, Radarr,
qBittorrent, VoidAuth, Immich, Framerr, WireGuard, PDS Bluesky, Kodi, gamescope, Moonlight).

## Prérequis

```bash
ansible-galaxy collection install -r requirements.yml
```

Adapte `inventory.ini` avec l'IP réelle du serveur, et `group_vars/all.yml` avec :
- la version de kernel-longterm à utiliser (vérifie sur le forum Fedora Discussion
  quelle branche est recommandée pour ta version de Fedora avant de lancer)
- le nom exact de ton pool ZFS
- les chemins de montage SSD/HDD si tu les changes

### Secrets (VoidAuth)

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml   # renseigne vault_db_password et vault_voidauth_storage_key
ansible-vault encrypt group_vars/all/vault.yml
```

**Important :** le fichier doit être sous `group_vars/all/`, pas directement sous
`group_vars/`. Ansible ne charge automatiquement que `group_vars/all.yml` (ou
`group_vars/all/*.yml`) et `group_vars/nas.yml` (ou `group_vars/nas/*.yml`, vu
le nom du groupe dans `inventory.ini`) — un fichier `group_vars/vault.yml` à
plat ne serait jamais lu, et `vault_db_password`/`vault_voidauth_storage_key`
resteraient indéfinies au moment de lancer le playbook.

Lance ensuite le playbook avec `--ask-vault-pass` (ou `--vault-password-file`).
Ne jamais committer `group_vars/all/vault.yml` en clair.

### Si tu as déjà un vault.yml chiffré (mise à jour du rôle immich)

Le rôle `immich` attend `vault_immich_db_password`, qui n'existe pas encore
si ton `group_vars/all/vault.yml` a été chiffré avant l'ajout de ce rôle.
Décrypte, ajoute la valeur, rechiffre :

```bash
ansible-vault decrypt group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml   # ajoute : vault_immich_db_password: "..."
ansible-vault encrypt group_vars/all/vault.yml
```

### Si tu as déjà un vault.yml chiffré (mise à jour du rôle pds)

Même chose pour le rôle `pds` (voir la section PDS plus bas pour le détail
des trois valeurs et les commandes pour les générer) : `vault_pds_jwt_secret`,
`vault_pds_admin_password` et `vault_pds_plc_rotation_key` n'existent pas
encore dans un vault chiffré avant l'ajout de ce rôle.

```bash
ansible-vault decrypt group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml   # ajoute les 3 valeurs vault_pds_*
ansible-vault encrypt group_vars/all/vault.yml
```

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

## SSH

Le rôle `ssh_hardening` tourne en tout premier (avant même `kernel_longterm`)
et désactive l'authentification par mot de passe (clés uniquement),
`PermitRootLogin`, et réduit `MaxAuthTries` à
`{{ ssh_hardening_max_auth_tries }}` (`group_vars/all/vars.yml`). Le jail
`fail2ban` sur sshd (voir plus bas) reste utile en complément, mais un
bruteforce ne devrait de toute façon jamais pouvoir essayer un mot de passe.

**Garde-fou intégré :** le rôle vérifie qu'une clé publique est déjà
enregistrée dans `authorized_keys` pour `{{ ansible_user }}` (celui
d'`inventory.ini`) *avant* de couper l'authentification par mot de passe -
si aucune clé n'est trouvée, le rôle échoue explicitement plutôt que de te
verrouiller dehors. Assure-toi donc d'avoir déjà déployé ta clé (`ssh-copy-id`
ou équivalent) avant le tout premier lancement du playbook.

## Samba et droits sur la médiathèque

Le rôle `samba` déploie un share `[Media]` dédié pointant sur
`{{ hdd_mountpoint }}/media` — le dossier partagé entre Sonarr, Radarr,
qBittorrent et Jellyfin. Le share `[SSD]` pointe uniquement sur
`{{ ssd_mountpoint }}/shared` (usage personnel), jamais sur
`{{ ssd_mountpoint }}` en entier : `appdata/` (secrets des containers,
certificats TLS internes de Caddy) et `podman/` (storage des containers)
ne sont donc jamais exposés par Samba.

Pour que les fichiers restent lisibles/modifiables des deux côtés (containers
*arr **et** Samba), deux mécanismes se combinent :

- `media_common` pose le bit setgid (`02775`) sur `tvshows/`, `movies/` et
  `downloads/` : tout fichier créé dedans hérite du groupe `svc-media`,
  peu importe qui l'a créé.
- Le share `[Media]` force ce même groupe côté Samba avec `force group =
  svc-media` — inutile donc d'ajouter les comptes de `samba_valid_users` au
  groupe UNIX `svc-media` pour que ça marche.

C'est aussi pour ça que `samba` tourne après `podman` (qui crée l'utilisateur
`svc-media`) et après `media_common` (qui crée les dossiers) dans `site.yml` —
inverser l'ordre n'empêcherait pas `smbd` de démarrer, mais autant garder les
dépendances explicites plutôt que de compter sur une résolution tardive côté
Samba.

Si tu ajoutes un nouvel utilisateur Samba : mets-le dans `samba_valid_users`
(`group_vars/all/vars.yml`) et dans `samba_accounts` (`group_vars/all/vault.yml`,
avec son mot de passe), puis relance le rôle avec `--tags samba`.

## Monitoring (Beszel)

Un hub central (sous `svc-monitor`, un utilisateur système dédié comme les
autres) + un agent par utilisateur de service (`svc-proxy`, `svc-sso`,
`svc-media`, `svc-photos`, `svc-monitor` lui-même, `svc-pds`). Ce n'est pas une
architecture choisie par goût de la complexité : un agent Beszel ne parle
qu'à un seul socket Podman à la fois, et avec Podman rootless cloisonné
par utilisateur (`containers_users`), un seul agent ne peut voir que les
containers de SON utilisateur. Chaque agent tourne donc en `Network=host`
(comme Caddy, et pour la même raison : atteindre le hub publié en loopback
sous un autre utilisateur) — conséquence assumée : tous les agents
remontent les mêmes métriques CPU/RAM/disque/réseau système, seule la
liste de containers Podman visibles diffère d'un agent à l'autre dans le
dashboard.

Le dashboard est exposé sur `monitor.{{ public_domain }}`, avec les deux
mêmes couches que les autres outils d'admin (`lan_only` + `voidauth_auth`)
— Beszel montre autant d'informations sensibles sur le système que
Cockpit (durci de la même façon, voir plus bas) et mérite la même
protection.

### Bootstrap manuel requis (une fois)

Comme pour VoidAuth (invitation), Beszel a besoin d'une étape manuelle
après le tout premier déploiement — pas d'automatisation fiable possible
ici : chaque agent a besoin d'un token d'enregistrement **unique** (en
réutiliser un seul pour deux agents casse la connexion du second, bug
connu de Beszel), et le "Universal Token" censé permettre
l'auto-enregistrement sans étape manuelle a des bugs d'enregistrement
documentés côté API. Autant suivre le chemin réellement supporté.

1. `ansible-playbook -i inventory.ini site.yml --tags beszel --ask-vault-pass`
   (avec `vault_beszel_tokens` encore vide dans `group_vars/all/vault.yml`
   — les agents démarreront mais resteront "déconnectés", c'est normal).
2. Ouvre `https://monitor.{{ public_domain }}` (ou
   `http://<IP du NAS>:{{ beszel_hub_port }}` en direct le temps du
   bootstrap, avant que le DNS/VoidAuth ne soient en place) et crée le
   compte admin du hub.
3. `Settings > Tokens` : copie la clé publique du hub affichée là dans
   `beszel_hub_key` (`group_vars/all/vars.yml` — **pas un secret**, c'est
   une clé publique, pas besoin du vault pour celle-là).
4. Pour **chacun** des 6 utilisateurs (`svc-proxy`, `svc-sso`,
   `svc-media`, `svc-photos`, `svc-monitor`, `svc-pds`) : bouton `Add System`,
   nomme-le comme l'utilisateur, copie le token généré dans
   `vault_beszel_tokens.<nom>` (`group_vars/all/vault.yml`).
5. Rechiffre le vault (`ansible-vault encrypt group_vars/all/vault.yml`),
   relance `--tags beszel` : les 6 agents se connectent avec leur vrai
   token.

## Dashboard (Framerr)

Dashboard homelab avec widgets (statut Jellyfin, calendriers Sonarr/Radarr,
requêtes Seerr, téléchargements qBittorrent, métriques système via
Glances...) et tabs iframe vers ces mêmes apps - voir
https://github.com/Framerrr/Framerr. Déployé sous `media_user`, sur le
même réseau Podman (`media.network`) que le reste de la stack média :
c'est ce qui lui permet de joindre les autres containers par leur nom
(`http://sonarr:8989`, `http://glances:61208`, etc.) une fois les
intégrations configurées dans son interface. Exposé sur
`framerr.{{ public_domain }}`, derrière les deux mêmes couches que les
autres outils d'admin (`lan_only` + `voidauth_auth`).

Le rôle `glances` (voir plus bas) n'existe que pour alimenter le widget
"Glances" natif de Framerr (CPU/mémoire/disque/réseau/température) - ce
n'est pas l'outil de monitoring principal du projet, ce rôle reste
`beszel` (historique, alerting, un agent par utilisateur de service).
Contrairement au reste de la stack, pas de vhost Caddy pour Glances :
Framerr le consomme en interne par nom de container
(`http://glances:61208`), et le port publié
(`roles/glances/templates/glances.container.j2`) reste en loopback
(`127.0.0.1:61208`) - accessible uniquement depuis le NAS lui-même
(navigateur local ou tunnel SSH), pas depuis le reste du LAN.

### Étape manuelle requise (une fois)

Contrairement à Beszel/VoidAuth, pas de bootstrap multi-étapes : juste une
clé à générer **avant** le premier déploiement.

1. `openssl rand -hex 32`, colle le résultat dans
   `vault_framerr_secret_encryption_key` (`group_vars/all/vault.yml`) -
   cette clé chiffre au repos les API keys/tokens que tu vas rentrer dans
   Framerr pour chaque intégration. La changer après coup rend illisibles
   les intégrations déjà configurées.
2. `ansible-vault encrypt group_vars/all/vault.yml`, puis
   `ansible-playbook -i inventory.ini site.yml --tags framerr --ask-vault-pass`.
3. Ouvre `https://framerr.{{ public_domain }}`, suis l'assistant de
   configuration initial, puis ajoute tes intégrations (Jellyfin, Sonarr,
   Radarr, Seerr, qBittorrent, Glances...) depuis son interface admin -
   leurs URLs internes utilisent le nom de container (`http://jellyfin:8096`,
   `http://sonarr:8989`, `http://glances:61208`, ...) puisqu'ils partagent
   tous `media.network`. Cette étape n'est pas automatisée par ce playbook
   (nécessiterait de stocker l'API key de chaque app dans le vault et
   d'appeler l'API interne de Framerr - pas fait pour l'instant).

## Réseau social (PDS Bluesky)

Serveur personnel de données (Personal Data Server) pour le protocole AT
(Bluesky) - voir https://github.com/bluesky-social/pds. Déployé sous
`pds_user` (`svc-pds`, utilisateur système dédié comme le reste de la
stack), stockage SQLite interne (pas de conteneur DB séparé comme pour
VoidAuth/Immich) : tout vit sous un seul dossier
`{{ ssd_mountpoint }}/appdata/pds/data` monté sur `/pds` dans le conteneur.

**C'est le seul service de tout ce projet qui casse sciemment le modèle du
reste de la stack.** Partout ailleurs, `{{ public_domain }}` est un TLD
`.internal` non routable, chaque vhost utilise `tls internal` (certificat
auto-signé de l'autorité interne de Caddy) et tout est protégé par
`lan_only` et/ou `voidauth_auth`. Le PDS ne peut suivre aucune de ces trois
règles :

- Il lui faut un **vrai nom de domaine public** (`pds_hostname`,
  `group_vars/all/vars.yml`), avec un vrai enregistrement DNS A/AAAA
  pointant vers l'IP publique de ce NAS, et les ports 80/tcp et 443/tcp de
  ce NAS redirigés depuis ton routeur (déjà ouverts côté firewalld par le
  rôle `caddy`, qui sert aussi ce vhost - la redirection sur le routeur
  lui-même n'est pas automatisable depuis ce playbook, comme pour le port
  WireGuard plus haut).
- Caddy lui obtient donc un **vrai certificat Let's Encrypt** (ACME
  HTTP-01) au lieu de `tls internal` - voir le vhost dédié dans
  `roles/caddy/templates/Caddyfile.j2`, seule exception du fichier.
- Il n'est protégé **ni par `lan_only` ni par `voidauth_auth`** : le
  protocole AT exige que ce endpoint soit joignable sans barrière depuis
  n'importe où sur internet (les autres PDS, le relay `bsky.network`,
  l'AppView et l'appli mobile Bluesky doivent tous pouvoir l'atteindre pour
  la fédération et la résolution du DID) - un `forward_auth` devant
  casserait l'accès API/mobile, exactement comme pour Immich. L'API
  d'administration du PDS (création de comptes/invitations) est protégée
  par son propre mot de passe (`PDS_ADMIN_PASSWORD`), indépendant de
  VoidAuth.

**Sauvegarde `{{ ssd_mountpoint }}/appdata/pds/data` régulièrement**, en
plus du reste : contrairement à la majorité des autres apps de ce projet
(où perdre l'appdata veut dire reconfigurer une intégration), perdre ce
dossier veut dire perdre la clé de rotation PLC de ton compte - sans elle,
plus aucun moyen de prouver que tu contrôles ton propre DID si tu dois un
jour migrer vers un autre PDS. C'est la seule copie qui existe.

### Étape manuelle requise (une fois, avant le premier déploiement)

Comme pour WireGuard, des secrets à préparer d'abord (ici trois, aucun
d'eux n'a de bootstrap web comme VoidAuth/Beszel) :

1. Génère les trois secrets et colle-les dans `group_vars/all/vault.yml`
   (voir aussi les commentaires dans `vault.yml.example`) :

   ```bash
   openssl rand -hex 16   # -> vault_pds_jwt_secret

   openssl ecparam --name secp256k1 --genkey --noout --outform DER \
     | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32
   # -> vault_pds_plc_rotation_key

   # vault_pds_admin_password : un mot de passe fort de ton choix
   ```

2. Renseigne `pds_hostname` (ton vrai domaine public) et `pds_admin_email`
   dans `group_vars/all/vars.yml`.
3. Chez ton registrar/DNS : un enregistrement A (et AAAA si IPv6) pour
   `pds_hostname` vers l'IP publique de ce NAS.
4. Sur ton routeur : redirige les ports 80/tcp et 443/tcp vers ce NAS
   (si ce n'est pas déjà fait pour Jellyfin/Seerr).
5. `ansible-vault encrypt group_vars/all/vault.yml`, puis
   `ansible-playbook -i inventory.ini site.yml --tags pds --ask-vault-pass`.
6. Vérifie que le certificat a bien été émis :
   `curl https://{{ pds_hostname }}/xrpc/_health` doit répondre
   `{"version":"..."}`.
7. `PDS_INVITE_REQUIRED=true` (inscriptions sur invitation, choisi pour ce
   déploiement) : crée d'abord un code d'invitation, puis un compte, avec
   les deux appels `curl` suivants (adaptés de `pdsadmin/account.sh` du
   dépôt officiel - ce playbook n'installe pas le script `pdsadmin`
   lui-même, seulement l'image du serveur) :

   ```bash
   # Récupère PDS_ADMIN_PASSWORD dans
   # {{ ssd_mountpoint }}/appdata/pds/pds.env (sous svc-pds sur le NAS)
   curl --request POST --user "admin:<PDS_ADMIN_PASSWORD>" \
     --header "Content-Type: application/json" \
     --data '{"useCount": 1}' \
     "https://{{ pds_hostname }}/xrpc/com.atproto.server.createInviteCode"
   # -> récupère le champ "code" de la réponse

   curl --request POST --header "Content-Type: application/json" \
     --data '{"email":"toi@example.com","handle":"toi.{{ pds_hostname }}","password":"un-mot-de-passe-de-compte","inviteCode":"<code ci-dessus>"}' \
     "https://{{ pds_hostname }}/xrpc/com.atproto.server.createAccount"
   ```

8. Dans l'appli Bluesky : écran de connexion -> "I already have an
   account" -> "Advanced" -> "Hosting provider" -> colle
   `https://{{ pds_hostname }}` -> connecte-toi avec le handle/mot de passe
   créés à l'étape précédente.

Point non testé en conditions réelles à vérifier après coup : le handle
`toi.{{ pds_hostname }}` (voir `PDS_SERVICE_HANDLE_DOMAINS`,
`roles/pds/templates/pds.env.j2`) suppose un DNS wildcard
(`*.{{ pds_hostname }}`) que ce playbook ne configure pas - sans lui, ce
handle précis ne résoudra pas correctement. Pour un usage perso à un seul
compte, plus simple d'utiliser un domaine que tu possèdes déjà comme handle
et de le vérifier via la méthode DNS TXT/well-known de Bluesky (Settings >
Handle > "I have my own domain" dans l'appli) plutôt que de dépendre du
sous-domaine `*.{{ pds_hostname }}`.

## VPN (WireGuard / wg-easy)

Accès distant aux services qui ne sont normalement joignables qu'en local
(127.0.0.1) ou depuis le LAN (`lan_only` dans le Caddyfile) - typiquement
Sonarr/Radarr/Prowlarr/qBittorrent/Framerr/Cockpit depuis l'extérieur, sans
les exposer directement sur internet un par un.

**Seule vraie exception rootful de tout ce projet.** Tout le reste tourne en
Podman rootless, cloisonné par utilisateur système (`containers_users`) -
wg-easy ne peut pas suivre ce même patron : créer l'interface `wg0` doit se
faire directement dans le netns de l'**hôte**, ce qu'un utilisateur non-root
ne peut structurellement pas faire (propriété des namespaces Linux - une
capability gagnée dans le user namespace d'un container rootless n'a
d'effet que sur les ressources possédées par CE namespace, jamais sur le
netns hôte). D'où un rôle à part (`roles/wireguard`), qui déploie un quadlet
**système** (`/etc/containers/systemd/`, géré par le systemd racine, pas
`--user`) plutôt que de passer par `quadlet_common`.

Cette exception est volontairement resserrée au minimum :
- `SYS_MODULE` n'est jamais accordé au container : le module `wireguard`
  est chargé une bonne fois pour toutes au boot par ce rôle
  (`community.general.modprobe`, persistant via `/etc/modules-load.d/`).
- Le port de l'UI web (`{{ wg_easy_ui_port }}`, 51821 par défaut) n'est
  **jamais** ouvert dans firewalld - exactement le même traitement que
  Cockpit. Seul Caddy l'atteint, en loopback, derrière les deux mêmes
  couches que les autres outils d'admin (`lan_only` + `voidauth_auth`, voir
  `vpn.{{ public_domain }}` dans le Caddyfile). Elle reste joignable une
  fois déjà connecté au VPN (le sous-réseau `10.10.0.0/24` est dans
  `private_ranges`, donc dans `lan_only`) - pas besoin d'être sur le LAN
  pour gérer tes pairs en déplacement.
- Seul le tunnel lui-même (`{{ wg_easy_port }}`/udp, 51820 par défaut) doit
  être redirigé depuis internet sur ton routeur - jamais le port de l'UI.
  Pas automatisable depuis ce playbook (dépend de ton matériel réseau).

### Étape manuelle requise (une fois, avant le premier déploiement)

Comme pour Framerr, une seule valeur à préparer avant de lancer le rôle -
mais ici il faut Podman déjà installé pour la générer (donc au moins un
premier `--tags podman` avant ceci) :

1. `podman run --rm -it ghcr.io/wg-easy/wg-easy:15 wgpw 'ton_mot_de_passe'`
   génère le hash bcrypt du mot de passe de connexion à l'UI - colle le
   résultat tel quel dans `vault_wg_easy_password_hash`
   (`group_vars/all/vault.yml`).
2. Renseigne aussi `wg_easy_host` (`group_vars/all/vars.yml`) avec ton IP
   publique ou ton nom de domaine dynamique (DuckDNS, Cloudflare DDNS...).
3. `ansible-vault encrypt group_vars/all/vault.yml`, puis
   `ansible-playbook -i inventory.ini site.yml --tags wireguard --ask-vault-pass`.
4. Redirige `{{ wg_easy_port }}` (udp) vers ce NAS sur ton routeur.
5. Ouvre `https://vpn.{{ public_domain }}` **depuis le LAN** (le premier
   accès ne peut pas passer par le VPN lui-même, logique), crée un premier
   pair, récupère son QR code ou son fichier `.conf`.

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
    `prowlarr.{{ public_domain }}`, `qbittorrent.{{ public_domain }}` et
    `framerr.{{ public_domain }}` passent par les DEUX couches : filtre LAN
    (`remote_ip private_ranges` + `abort`,
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
- Avant chaque exécution qui touche au kernel, vérifie manuellement la
  compatibilité ZFS avec la nouvelle version sur
  https://github.com/openzfs/zfs/releases
- Cockpit (installé via `cockpit-podman`, dépendance du rôle `podman`) est
  désormais derrière les mêmes couches que le reste (`lan_only` +
  `voidauth_auth`, via `cockpit.{{ public_domain }}`) : `cockpit.socket`
  est replié sur `127.0.0.1:9090` (drop-in systemd, voir
  `roles/zfs/tasks/main.yml` — regroupé avec le plugin ZFS existant qui le
  touchait déjà) et retiré de firewalld. Un point non testé en conditions
  réelles à vérifier après le premier déploiement : ouvre
  `https://cockpit.{{ public_domain }}` et confirme que la page de login
  s'affiche sans erreur liée à l'en-tête `Origin` dans la console du
  navigateur (`roles/zfs/templates/cockpit.conf.j2` contient le détail du
  réglage et le lien vers la doc Cockpit si besoin d'ajuster).
- Le monitoring (`beszel`) nécessite un bootstrap manuel après le premier
  déploiement — voir la section Monitoring plus haut.
- `wireguard` (wg-easy) nécessite un secret préparé avant le premier
  déploiement — voir la section VPN plus haut. Point non testé en
  conditions réelles à vérifier après coup : la résolution DNS de
  `*.{{ public_domain }}` une fois connecté au VPN dépend de `wg_easy_dns`
  (`group_vars/all/vars.yml`) — pointe-le vers un DNS local qui connaît déjà
  `public_domain` si tu en as un (Pi-hole/AdGuard), sinon prévois un
  `/etc/hosts` par appareil client, rien ne le fait automatiquement.
- `AutoUpdate=registry` est actif sur tous les quadlets (mis à jour
  automatiquement par `podman-auto-update.timer`). Caddy et VoidAuth ont
  un `HealthCmd` explicite (voir `roles/caddy/templates/healthcheck.sh.j2`
  et `roles/voidauth/templates/healthcheck.sh.j2`) pour que Podman puisse
  faire un rollback automatique si une mise à jour casse l'image — ce sont
  les deux seuls points de passage obligés de toute la stack (reverse
  proxy unique + SSO unique), une image cassée sans filet de sécurité y
  couperait l'accès à tout le monde. Les autres apps (Jellyfin, *arr,
  Seerr, qBittorrent) n'ont pas encore ce filet : moins critique
  individuellement (une seule app en panne, pas toute la stack), mais ça
  reste une amélioration possible si tu veux généraliser le principe.
- `pds` (PDS Bluesky) nécessite trois secrets préparés avant le premier
  déploiement - voir la section Réseau social plus haut. C'est aussi le
  seul rôle du projet à exposer un vrai domaine public avec une vraie
  certif Let's Encrypt : la redirection des ports 80/443 sur le routeur et
  le DNS public de `pds_hostname` restent entièrement manuels. Le handle
  `*.{{ pds_hostname }}` (sous-domaine) n'est pas garanti fonctionner sans
  DNS wildcard, non géré par ce playbook - voir la section Réseau social
  pour l'alternative recommandée.
- qBittorrent : ce playbook ne force aucun mot de passe WebUI. Les
  versions récentes de l'image génèrent un mot de passe temporaire
  aléatoire au premier démarrage et l'affichent dans les logs
  (`podman logs --user qbittorrent` sous `svc-media`, ou
  `journalctl --user -u qbittorrent.service`) plutôt que d'utiliser un
  couple identifiants par défaut fixe — récupère-le là et change-le dès
  la première connexion à `qbittorrent.{{ public_domain }}`.

## Structure du projet

- `roles/quadlet_common/tasks/deploy.yml` : logique commune de déploiement
  d'un quadlet (UID/GID, dossiers appdata, fichiers d'environnement secrets,
  daemon-reload, start/enable), incluse par tous les rôles applicatifs
  Podman (`jellyfin`, `sonarr`, `radarr`, `prowlarr`, `seerr`, `qbittorrent`,
  `framerr`, `glances`, `caddy`, `voidauth`, `media_common`, `beszel`,
  `pds`) plutôt que dupliquée dans
  chacun. Un nouveau service = un nouveau rôle avec un `tasks/main.yml` de
  quelques lignes + ses templates, pas un copier-coller des 6 tâches
  UID/dossiers/quadlet/reload/start. `beszel` est le seul à boucler
  l'include lui-même (un agent par utilisateur de service) plutôt que de
  l'appeler une seule fois — voir le commentaire dans
  `roles/beszel/tasks/main.yml`.
- `requirements.yml` : versions planchers des collections utilisées
  (`community.general`, `ansible.posix`).
- `ansible.cfg` : config du projet (inventaire par défaut, pas de fichiers
  `.retry`).
- `.gitignore` : bloque les fichiers qui ne doivent jamais être commit
  (vault en clair, mot de passe du vault, artefacts locaux).
- `.yamllint` : config yamllint du projet (voir commentaires dans le
  fichier pour les deux règles assouplies et pourquoi).
- `.github/workflows/lint.yml` : CI minimaliste (ansible-lint profil
  production + yamllint + syntax-check), aucun déploiement réel.

    sudo journalctl _SYSTEMD_USER_UNIT=caddy.service -n 50 --no-pager