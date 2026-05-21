---
title: Installation MikroTik
description: Guide complet pour installer Wima Zone Billing en container sur MikroTik RouterOS v7
---

# Installation sur MikroTik

Ce guide couvre l'installation **pas-à-pas** de Wima Zone Billing en mode container sur un routeur MikroTik RouterOS v7.

::: tip Avant de commencer
Assurez-vous d'avoir validé toute la [checklist de prérequis](/docs/guide/installation#requirements) : RouterOS v7.10+, USB ext4, licence WimaZone ITDevSuccess.
:::

## <Icon name="Router" color="warning" /> Routeurs compatibles

| Modèle | Architecture | RAM | Compatibilité |
|---|---|---:|---|
| L009UiGS-2HaxD-IN | ARM 32 bits (armv7) | 512 MB | Compatible |
| L009UiGS-RM | ARM 32 bits (armv7) | 512 MB | Compatible |
| hAP ax3 | ARM 64 bits | 1 GB | Compatible |
| RB4011 | ARM 32 bits (armv7) | 1 GB | Compatible |
| RB5009 | ARM 64 bits | 1 GB | Compatible |

Les routeurs MikroTik compatibles sont disponibles à l'achat sur la boutique en ligne : **[wimazone.mg/boutique](https://wimazone.mg/boutique)**

## <Icon name="Package" color="info" /> Cibles d'image

Image publique : `wimazone/billing:latest` (Docker Hub) — manifest multi-arch Alpine + MariaDB.

| Cible | Routeurs |
|---|---|
| `linux/arm/v7` | L009, RB4011, hAP ac² |
| `linux/arm64` | hAP ax³, RB5009, CCR2004/2116 |
| `linux/amd64` | serveur / CasaOS |

Tu n'as rien à spécifier côté MikroTik : l'engine container pull le variant correspondant à son architecture déclarée.

::: danger Modèles non supportés

**CPU EN7562CT (arm32v5 only)** — [doc officielle MikroTik](https://help.mikrotik.com/docs/display/ROS/Container#Container-Requirements) :
> For devices with EN7562CT CPU like the hEX Refresh, only arm32v5 container images are supported.

| Modèle | Raison |
|---|---|
| hEX refresh (**E50UG**) | Sandbox arm32v5 soft-float uniquement, incompatible Alpine musl armhf |
| hEX S 2025 (**E60iUGS**) | Même CPU EN7562CT, même restriction |

Pour ces modèles, installer [**wimalite**](/docs/guide/wimalite) (version PHP pur multi-arch qui inclut arm/v5) à la place.

**CPU MIPS (pas d'image PHP multi-arch)** :

| Modèle | Raison |
|---|---|
| hEX original (RB750Gr3) | MT7621A MIPS-BE + 256 Mo RAM |
| hEX S original (RB760iGS, 2018) | MT7621A MIPS-BE + 256 Mo RAM |
| hAP ac lite, hAP lite | MIPS + RAM très limitée |

Attention à ne pas confondre les noms : **hEX refresh ≠ hEX**, **hEX S 2025 ≠ hEX S original**, mais aucun n'est supporté (pour des raisons différentes). Prendre un L009, hAP ax³ ou RB5009 à la place.
:::

---

## 1) Vérifier le device-mode

Certains routeurs sortent d'usine avec `mode=home`, ce qui **désactive** la fonctionnalité container. Vérifier d'abord :

```routeros
/system/device-mode/print
```

Si `mode` est différent de `advanced`, basculer :

```routeros
/system/device-mode/update mode=advanced
```

::: warning Confirmation physique requise
RouterOS vous demandera d'**appuyer sur le bouton reset physique** du routeur dans les 60 secondes pour confirmer le changement. Sans cette confirmation, la commande est annulée.
:::

## 2) Activer le support container

```routeros
/system/device-mode/update container=yes
```

Même principe : une confirmation par bouton reset est demandée. Vérifier ensuite :

```routeros
/system/device-mode/print
# doit afficher : container: yes
```

## 3) Configurer le sous-système container

Par défaut, RouterOS utilise la RAM pour les fichiers temporaires et les couches d'image — ce qui sature très vite sur MikroTik. Il faut rediriger vers l'USB :

```routeros
/container/config/set \
  registry-url=https://registry-1.docker.io \
  tmpdir=usb1/tmp \
  layer-dir=usb1/layer
```

Vérifier :

```routeros
/container/config/print
```

## 4) Créer le bridge des containers

```routeros
/interface/bridge/add name=dockers comment="Bridge containers Wima Zone"
/ip/address/add address=172.17.0.1/24 interface=dockers comment="Gateway bridge containers"
```

Ajouter le bridge à la liste d'interfaces **LAN** pour que les règles firewall et DNS internes s'appliquent :

```routeros
/interface/list/member/add list=LAN interface=dockers
```

## 5) Créer l'interface VETH du container billing

```routeros
/interface/veth/add name=veth-billing address=172.17.0.2/24 gateway=172.17.0.1 comment="Wima Zone"
/interface/bridge/port/add bridge=dockers interface=veth-billing
```

## 6) Ajouter le NAT sortant des containers

```routeros
/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="Containers Internet"
```

## 7) Règles firewall (redirection portail + accès admin)

Rediriger le port externe **8080** du routeur vers le port 80 du container (c'est ainsi que les clients du LAN atteignent le portail Wima Zone) :

```routeros
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8080 action=dst-nat \
  to-addresses=172.17.0.2 to-ports=80 comment="Redirection portail Wima Zone"
```

Autoriser les ports entrants nécessaires (chain `input`) :

```routeros
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8080 action=accept comment="Portail Wima Zone"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8291 action=accept comment="Winbox"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8728 action=accept comment="API MikroTik"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8729 action=accept comment="API-SSL MikroTik"
```

::: warning Container → RouterOS REST API (obligatoire pour la licence)
Le container interroge RouterOS sur les ports **80** (HTTP) et **443** (HTTPS) pour lire le serial matériel via REST API (cf. [section 10 : licence](#_10-variables-d-environnement-du-container)). Si le bridge `dockers` n'est pas dans `interface-list=LAN` (ou si une règle `drop !LAN` filtre l'input), ajoutez explicitement :

```routeros
/ip/firewall/filter/add chain=input action=accept \
  src-address=172.17.0.0/24 \
  protocol=tcp dst-port=80,443 \
  place-before=[find where chain=input action=drop in-interface-list="!LAN"] \
  comment="Container Wima Zone -> RouterOS REST API"
```

Sans cette règle, le container échouera au boot avec `curl-error-7` ou `curl-error-28` lors de la lecture du serial, puis `ERREUR: fingerprint invalide ou vide` — boot loop garanti.
:::

::: tip Ordre des règles
Placez ces règles **avant** toute règle `drop` globale du chain `input`. Sinon elles seront ignorées. Utilisez `/ip/firewall/filter/move` si besoin, ou le `place-before=` ci-dessus qui le fait automatiquement.
:::

## 8) Configurer le DNS du routeur

```routeros
/ip/dns/set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes
```

## 9) Créer le stockage persistant MariaDB

L'image embarque **MariaDB** ; il faut persister son répertoire de données sur l'USB pour survivre aux redémarrages / mises à jour.

```routeros
/container/mounts/add src=usb1/billing-data dst=/data list=billing-db
```

::: danger `dst=` doit être `/data` (PAS `/var/lib/mysql`)
L'image utilise `/data/mysql` comme datadir MariaDB **à l'intérieur** d'un mount persistant `/data`. Le mount sert aussi à stocker `/data/.fingerprint` (licence).

✅ OK : `src=usb1/billing-data` + `dst=/data`
❌ KO : `src=usb1/billing-data/mysql` + `dst=/var/lib/mysql` (datadir ignoré → DB neuve à chaque pull, migrations rejouées, données perdues)
:::

::: warning Clé USB ext4 obligatoire
Les mounts container ne fonctionnent qu'avec un stockage formaté **ext4**. Vérifier avec `/disk/print` que le device `usb1` est bien reconnu. MariaDB refuse de démarrer sur FAT32/NTFS.
:::

::: danger `src=` doit être EN DEHORS de `root-dir`
Si tu utilises `root-dir=usb1/wimazone`, alors `src=` doit pointer vers un dossier **frère** (ex `usb1/billing-data`), **pas un sous-dossier** de `root-dir`. Sinon le mount est wipé au prochain `/container/remove` + repull — tu perds MariaDB **et** le fingerprint de licence, et le routeur sera rejeté HTTP 403 au prochain boot.

✅ OK : `root-dir=usb1/wimazone` + `src=usb1/billing-data`
❌ KO : `root-dir=usb1/wimazone` + `src=usb1/wimazone/billing-data`
:::

## 10) Variables d'environnement du container

```routeros
/container/envs/add list=billing-env key=APP_ENV value=production
/container/envs/add list=billing-env key=APP_DEBUG value=false
/container/envs/add list=billing-env key=DB_CONNECTION value=mysql
/container/envs/add list=billing-env key=DB_HOST value=127.0.0.1
/container/envs/add list=billing-env key=DB_PORT value=3306
/container/envs/add list=billing-env key=DB_DATABASE value=wimazone
/container/envs/add list=billing-env key=DB_USERNAME value=wimazone
/container/envs/add list=billing-env key=SYNC_ENABLED value=true
/container/envs/add list=billing-env key=OFFLINE_FALLBACK value=true
/container/envs/add list=billing-env key=MIKROTIK_API_HOST value=172.17.0.1
/container/envs/add list=billing-env key=MIKROTIK_API_USER value=admin
/container/envs/add list=billing-env key=MIKROTIK_API_PASSWORD value=VOTRE_PASSWORD_ADMIN_ROUTEROS
/container/envs/add list=billing-env key=LARAVEL_AUTO_MIGRATION value=true
/container/envs/add list=billing-env key=LARAVEL_AUTO_MIGRATION_OPTIONS value=--force
/container/envs/add list=billing-env key=LARAVEL_AUTO_STORAGE_LINK value=true
/container/envs/add list=billing-env key=LARAVEL_ENABLE_QUEUE_WORKER value=true
/container/envs/add list=billing-env key=LARAVEL_ENABLE_SCHEDULER value=true
/container/envs/add list=billing-env key=LARAVEL_QUEUE_WORKER_OPTIONS value="--queue=mikrotik,default --tries=1 --timeout=1200 --sleep=2"
/container/envs/add list=billing-env key=REDIRECT_URL value=http://votre-portail.wifi
/container/envs/add list=billing-env key=HOTSPOT_STATUS_TIMEOUT_SECONDS value=2
/container/envs/add list=billing-env key=HOTSPOT_STATUS_CACHE_SECONDS value=3
/container/envs/add list=billing-env key=HOTSPOT_STATUS_FAILURE_COOLDOWN_SECONDS value=20
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC value=true
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC_ONLY_ONLINE value=true
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC_PROCESS_NOW value=true
/container/envs/add list=billing-env key=LARAVEL_ENABLE_REVERB value=false
/container/envs/add list=billing-env key=BROADCAST_CONNECTION value=log
```

::: info Licence
`WIMAZONE_LICENSE_KEY` t'est envoyée **par email** par ITDevSuccess après l'achat. Ajoute-la à ton `billing-env` :

```routeros
/container/envs/add list=billing-env key=WIMAZONE_LICENSE_KEY value=<clé reçue par email>
```

Chaque routeur a sa propre clé, révocable individuellement depuis le portail admin.
:::

::: tip Reverb WebSocket (cyber-cafés uniquement)
Désactivé par défaut (`LARAVEL_ENABLE_REVERB=false`) car non utilisé en mode hotspot classique. Pour les **cyber-cafés** qui ont besoin du temps-réel (unlock instantané du poste, monitoring sessions actives), activer :

```routeros
/container/envs/set [find list="billing-env" key="LARAVEL_ENABLE_REVERB"] value=true
# Exposer le port WebSocket 8081 vers le LAN :
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8081 action=dst-nat \
  to-addresses=172.17.0.2 to-ports=8081 comment="Reverb WebSocket"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8081 action=accept comment="Reverb"
/container/stop [find name="Wima Zone"]
/container/start [find name="Wima Zone"]
```

Port 8081 (et non 8080) pour éviter le conflit avec le port du portail captif.
:::

::: warning Identité matérielle anti-fraude
La licence est **liée au serial matériel** du routeur (1 seat = 1 MikroTik). Au premier boot, le container interroge RouterOS via REST API (`https://172.17.0.1/rest/system/routerboard`) avec les credentials `MIKROTIK_API_USER` / `MIKROTIK_API_PASSWORD` pour lire le serial directement depuis le matériel — **impossible à falsifier**.

Le fingerprint envoyé au serveur de licences devient `serial-<SN>`, **stable au repull / reboot / réinstall RouterOS**. Un autre routeur tentant la même licence sera rejeté (HTTP 403).

::: tip Activer le service www sur MikroTik
```routeros
/ip/service/enable www
# Optionnel mais recommandé : restreindre à 172.17.0.0/24
/ip/service/set www address=172.17.0.0/24
```

⚠️ Activer le service ne suffit pas si le firewall bloque l'input depuis le bridge container. Voir la [règle `chain=input` pour `172.17.0.0/24` → ports 80/443](#_7-regles-firewall-redirection-portail-acces-admin) en section 7.
:::
:::

## 11) Créer le container Wima Zone

```routeros
/container/add \
  name="Wima Zone" \
  remote-image=wimazone/billing:latest \
  interface=veth-billing \
  root-dir=usb1/wimazone \
  mounts=billing-db \
  envlist=billing-env \
  start-on-boot=yes \
  logging=yes
```

## 12) Démarrer le container

```routeros
/container/start [find where name="Wima Zone"]
```

## 13) Vérifier les logs

```routeros
/container/log print follow where container="Wima Zone"
```

Durée du premier boot selon le matériel :

| Modèle | Premier boot | Reboots suivants |
|---|---|---|
| hAP ax³ / RB5009 | 2-3 min | 30 s |
| L009 | 3-5 min | 45 s |

Vous devriez voir à la fin :

```text
[scheduler] started
[queue] worker listening on mikrotik,default
[nginx] listening on 0.0.0.0:80
```

## 14) Walled Garden recommandé {#walled-garden}

### Walled Garden IP

```routeros
/ip/hotspot/walled-garden/ip/add dst-address=172.17.0.2 action=accept comment="Wima Zone container"
/ip/hotspot/walled-garden/ip/add dst-address=192.168.88.1 protocol=tcp dst-port=8291 action=accept comment="Winbox"
/ip/hotspot/walled-garden/ip/add dst-address=192.168.88.1 protocol=tcp dst-port=80 action=accept comment="WebFig HTTP"
/ip/hotspot/walled-garden/ip/add dst-address=192.168.88.1 protocol=tcp dst-port=443 action=accept comment="WebFig HTTPS"
```

### Walled Garden host

```routeros
/ip/hotspot/walled-garden/add dst-host=wimazone.wifi action=allow
/ip/hotspot/walled-garden/add dst-host=*.wimazone.wifi action=allow
/ip/hotspot/walled-garden/add dst-host=portal.wimazone.wifi action=allow
/ip/hotspot/walled-garden/add dst-host=wimazone.mg action=allow
/ip/hotspot/walled-garden/add dst-host=*.wimazone.mg action=allow
/ip/hotspot/walled-garden/add dst-host=wimacloud.mg action=allow
/ip/hotspot/walled-garden/add dst-host=*.wimacloud.mg action=allow
/ip/hotspot/walled-garden/add dst-host=itdevsuccess.com action=allow
/ip/hotspot/walled-garden/add dst-host=*.itdevsuccess.com action=allow
/ip/hotspot/walled-garden/add dst-host=embed.tawk.to action=allow
/ip/hotspot/walled-garden/add dst-host=va.tawk.to action=allow
/ip/hotspot/walled-garden/add dst-host=*.tawk.to action=allow
/ip/hotspot/walled-garden/add dst-host=api.befiana.cloud action=allow
/ip/hotspot/walled-garden/add dst-host=*.befiana.cloud action=allow
```

### Ports courants à autoriser

```routeros
/ip/hotspot/walled-garden/add dst-port=8728 action=allow comment="API MikroTik"
/ip/hotspot/walled-garden/add dst-port=8291 action=allow comment="Winbox"
```

## 15) Vérifications post-installation

```routeros
/interface/veth/print
/container/mounts/print
/container/envs/print
/ip/hotspot/walled-garden/print
/ip/hotspot/walled-garden/ip/print
/container/print
```

---

## <Icon name="LogIn" color="success" /> Premier accès {#premier-acces}

Une fois le container démarré, accédez au portail admin :

**URL :** `http://172.17.0.2` (depuis un appareil dans le réseau interne).

**Identifiants par défaut** (à changer immédiatement) :

```text
Email    : admin@wimazone.local
Password : ChangeMe!2026
```

::: danger Première action obligatoire

1. Connectez-vous avec le compte super-admin.
2. Changez le mot de passe depuis **Profil → Sécurité**.
3. Renseignez les APIs MVola / Befiana dans **Paramètres → APIs**.
4. Ajoutez votre routeur MikroTik dans **Paramètres → Routeurs**.
:::

## <Icon name="Database" color="primary" /> Backup & restore {#backup-restore}

### Sauvegarder la base MariaDB

Ouvrir un shell dans le container et exporter via `mysqldump` :

```routeros
/container/shell [find where name="Wima Zone"]
```

Puis, dans le shell du container :

```bash
mysqldump -u root -p"$MARIADB_ROOT_PASSWORD" wimazone > /data/mysql/backups/wimazone-$(date +%F).sql
```

Le fichier `.sql` est accessible depuis RouterOS dans `usb1/billing-data/mysql/backups/`. Pour le copier vers un dossier de sauvegarde dédié :

```routeros
/file/copy src=usb1/billing-data/mysql/backups/wimazone-YYYY-MM-DD.sql dst=usb1/backup/wimazone-YYYY-MM-DD.sql
```

Ou récupérer depuis un host distant via SCP :

```bash
scp admin@192.168.88.1:/usb1/backup/wimazone-2026-04-24.sql ./
```

### Restaurer une sauvegarde

```routeros
/container/shell [find where name="Wima Zone"]
```

Dans le shell du container :

```bash
mysql -u root -p"$MARIADB_ROOT_PASSWORD" wimazone < /data/mysql/backups/wimazone-YYYY-MM-DD.sql
```

::: tip Automatisation
Ajoutez un scheduler RouterOS qui déclenche un dump quotidien via le container :

```routeros
/system/scheduler/add name=billing-backup interval=1d start-time=03:00 on-event={
  /container/shell [find where name="Wima Zone"] command="sh -c 'mysqldump -u root -p\"\$MARIADB_ROOT_PASSWORD\" wimazone > /data/mysql/backups/wimazone-daily.sql'"
}
```

:::

## <Icon name="RefreshCw" color="info" /> Mise à jour {#mise-a-jour}

Les mises à jour se font en deux étapes : récupération de la nouvelle image et recréation du container. Les migrations Laravel s'exécutent automatiquement au redémarrage (`LARAVEL_AUTO_MIGRATION=true`).

```routeros
# 1. Arrêter et supprimer le container actuel (les données USB sont conservées)
/container/stop [find where name="Wima Zone"]
/container/remove [find where name="Wima Zone"]

# 2. Recréer avec la même configuration
/container/add \
  name="Wima Zone" \
  remote-image=wimazone/billing:latest \
  interface=veth-billing \
  root-dir=usb1/wimazone \
  mounts=billing-db \
  envlist=billing-env \
  start-on-boot=yes \
  logging=yes

# 3. Démarrer
/container/start [find where name="Wima Zone"]
```

::: warning Toujours sauvegarder avant
Faites un dump MariaDB (cf. section précédente) **avant** toute mise à jour.
:::

## <Icon name="Wrench" color="warning" /> Dépannage {#depannage}

### Le container ne démarre pas

```routeros
/container/log print where container="Wima Zone"
```

Symptômes courants :

| Message | Cause probable | Solution |
|---|---|---|
| `exited with signal 4 (Illegal instruction)` | Routeur avec CPU EN7562CT (hEX refresh / hEX S 2025) qui réclame `archVariant:v5` → sandbox MikroTik restreint à arm32v5 soft-float, incompatible avec Alpine armhf | Ces modèles ne sont pas supportés. Utiliser un L009, hAP ax³ ou RB5009 |
| `SIGKILL` / `OOMKilled` | Manque de RAM | Réduire les workers queue, utiliser modèle ax³ |
| `licence rejetee (HTTP 401/403)` | Licence invalide, révoquée, ou seat déjà pris par un autre fingerprint | Voir section [Repull a wipé /data](#repull-fingerprint-perdu) ci-dessous |
| `impossible de lire le serial via RouterOS REST API` | Credentials `MIKROTIK_API_*` invalides ou service `www` désactivé | `/ip/service/enable www` + vérifier user/password admin RouterOS |
| `Can't connect to MySQL server on '127.0.0.1'` | MariaDB pas encore prête | Attendre 30 s après le démarrage ; vérifier `s6-svstat mariadb` |
| `Access denied for user 'wimazone'` | Mot de passe DB incorrect | Vérifier `DB_PASSWORD` et `MARIADB_ROOT_PASSWORD` |
| `Unknown database 'wimazone'` | Mount `/data` vide ou corrompu | Vérifier `src=usb1/billing-data dst=/data`, recréer le mount |
| Migrations rejouées + DB neuve à chaque pull | Mount avec mauvais `dst=` (ex `/var/lib/mysql`) — datadir MariaDB jamais persisté | Recréer le mount avec `dst=/data` (cf section 9) |
| `502 Bad Gateway` | PHP-FPM saturé | Baisser `HOTSPOT_STATUS_TIMEOUT_SECONDS` à 1 |
| `max_children reached` | Trop de requêtes simultanées | Garder `MIKROTIK_BOOT_HOTSPOT_SYNC_PROCESS_NOW=false` |

### Repull a wipé /data — licence rejetée HTTP 403 {#repull-fingerprint-perdu}

**Symptôme** au boot après `/container/remove` + repull :

```
[startup] ERREUR: licence rejetee (HTTP 403) — verifier WIMAZONE_LICENSE_KEY
[startup] ERREUR: licence invalide — pas de fallback offline pour ce cas.
```

**Cause** : le mount `/data` n'était pas correctement persistant (voir [warning section 9](#9-créer-le-stockage-persistant-mariadb)). Le fichier `/data/.fingerprint` a été perdu. Si le fingerprint était un UUID aléatoire (ancien client < v4.10.2), l'ancien usage occupe encore le seat côté serveur → le nouveau est rejeté.

**Recovery** (à faire 1 fois) :

1. **Côté wimazone.mg/admin/licenses** : trouver la licence, identifier l'ancien usage (`uuid-...`), cliquer **« Libérer »** pour révoquer le seat.
2. **Sur le MikroTik**, vérifier que les credentials API sont configurés :

   ```routeros
   /ip/service/enable www
   /container envs add list=billing-env key=MIKROTIK_API_USER value=admin
   /container envs add list=billing-env key=MIKROTIK_API_PASSWORD value=<pass>
   /container stop  [find name=billing]
   /container start [find name=billing]
   ```

3. **Au prochain boot**, le container lit directement le serial via REST API et l'enregistre côté serveur. Cette identité matérielle est **stable au repull, reboot, et même réinstall RouterOS** — le problème ne se reproduira plus.

4. **Corriger le mount** pour aussi sauver MariaDB :

   ```routeros
   /container/mounts/add src=usb1/billing-data dst=/data list=billing-db
   # src= EN DEHORS de root-dir (cf section 9)
   ```

### Diagnostics réseau (VETH / bridge)

```routeros
/interface/veth/print detail
/interface/bridge/port/print where bridge=dockers
/ip/address/print where interface=dockers
/ping 172.17.0.2 count=4
```

Si `172.17.0.2` ne répond pas, vérifier que le container est bien attaché au VETH :

```routeros
/container/print detail
```

### Synchronisation hotspot bloquée

```routeros
/container/shell [find where name="Wima Zone"]
# Dans le container :
php artisan hotspot:sync --router=1 --dry-run
```

### Logs Laravel

Depuis un shell ouvert dans le container :

```bash
tail -f /var/www/html/storage/logs/laravel.log
```

### API MikroTik inaccessible

Vérifier les credentials et le port :

```routeros
/user/print
/ip/service/print
# API doit être activée (port 8728) ou API-SSL (8729)
/ip/service/enable api
```

---

## <Icon name="BookOpen" color="info" /> Notes d'exploitation

- `OFFLINE_FALLBACK=true` : le container démarre avec le code local si l'API wimazone est indisponible.
- Mount `/data` sur `usb1/billing-data` : les données MariaDB (`/data/mysql`) et le fingerprint licence (`/data/.fingerprint`) survivent à la recréation du container.
- `LARAVEL_AUTO_STORAGE_LINK=false` : évite un blocage au boot sur certains mounts.
- Préparer les assets frontend en amont (pas de build frontend lourd sur MikroTik).
- Pour surveiller en continu : `/container/log print follow where container="Wima Zone"`.
