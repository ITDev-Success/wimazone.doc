---
title: Fametrahana MikroTik
description: Torolàlana feno hametrahana ny Wima Zone Billing amin'ny container MikroTik RouterOS v7
---

# Fametrahana amin'ny MikroTik

Ity torolàlana ity dia mikasika ny fametrahana **dingana tsirairay** ny Wima Zone Billing amin'ny mode container amin'ny routeur MikroTik RouterOS v7.

::: tip Alohan'ny hanombohana
Hamarino fa vita daholo ny [lisitra fepetra takiana](/mg/docs/guide/installation#requirements) : RouterOS v7.10+, USB ext4, licence WimaZone ITDevSuccess.
:::

## <Icon name="Router" color="warning" /> Routeur mifanaraka

| Modely | Architecture | RAM | Fitoviana |
|---|---|---:|---|
| L009UiGS-2HaxD-IN | ARM 32 bits (armv7) | 512 MB | Mifanaraka |
| L009UiGS-RM | ARM 32 bits (armv7) | 512 MB | Mifanaraka |
| hAP ax3 | ARM 64 bits | 1 GB | Mifanaraka |
| RB4011 | ARM 32 bits (armv7) | 1 GB | Mifanaraka |
| RB5009 | ARM 64 bits | 1 GB | Mifanaraka |

Azo vidiana ao amin'ny fivarotana an-tserasera : **[wimazone.mg/boutique](https://wimazone.mg/boutique)**

## <Icon name="Package" color="info" /> Sary kendrena

Sary ho an'ny besinimaro : `wimazone/billing:latest` (Docker Hub) — manifest multi-arch Alpine + MariaDB.

| Kendrena | Routeur |
|---|---|
| `linux/arm/v7` | L009, RB4011, hAP ac² |
| `linux/arm64` | hAP ax³, RB5009, CCR2004/2116 |
| `linux/amd64` | serveur / CasaOS |

Tsy misy kajy manokana ny MikroTik : ny engine container dia maka ny variant mifanaraka amin'ny architecture hita.

::: danger Modely tsy tohanana

**CPU EN7562CT (arm32v5 ihany)** — [doc ofisialin'ny MikroTik](https://help.mikrotik.com/docs/display/ROS/Container#Container-Requirements) :
> For devices with EN7562CT CPU like the hEX Refresh, only arm32v5 container images are supported.

| Modely | Antony |
|---|---|
| hEX refresh (**E50UG**) | Sandbox arm32v5 soft-float ihany, tsy mifanaraka Alpine musl armhf |
| hEX S 2025 (**E60iUGS**) | Mitovy CPU EN7562CT, fetra mitovy |

Ho an'ireo modely ireo, apetraho [**wimalite**](/mg/docs/guide/wimalite) (version PHP madio multi-arch izay mampiditra arm/v5) eo amin'ny toerany.

**CPU MIPS (tsy misy sary PHP multi-arch)** :

| Modely | Antony |
|---|---|
| hEX taloha (RB750Gr3) | MT7621A MIPS-BE + 256 Mo RAM |
| hEX S taloha (RB760iGS, 2018) | MT7621A MIPS-BE + 256 Mo RAM |
| hAP ac lite, hAP lite | MIPS + RAM kely be |

Tandremo tsy hafangaroana ny anarana : **hEX refresh ≠ hEX**, **hEX S 2025 ≠ hEX S taloha**, fa tsy tohanana izy ireo rehetra (antony samihafa). Aleo maka L009, hAP ax³ na RB5009.
:::

---

## 1) Hamarino ny device-mode

Ny routeur sasany dia `mode=home` rehefa avy amin'ny orinasa, ka **tsy mandeha** ny container. Hamarino aloha :

```routeros
/system/device-mode/print
```

Raha tsy `advanced` ny `mode`, ovao :

```routeros
/system/device-mode/update mode=advanced
```

::: warning Porofo fizaràna ilaina
RouterOS dia hangataka anao **hanindry ny bouton reset** an'ny routeur ao anatin'ny 60 segondra mba hanamarinana ny fiovana. Raha tsy izany dia tsy mandeha ny baiko.
:::

## 2) Alefaso ny fanohanana container

```routeros
/system/device-mode/update container=yes
```

Mitovy amin'iny : hangataka fanamarinana amin'ny bouton reset ihany koa. Avy eo hamarino :

```routeros
/system/device-mode/print
# tokony hiseho : container: yes
```

## 3) Apetraho ny rafitra container

Rehefa tsy voaova, RouterOS dia mampiasa RAM ho an'ny rakitra vonjimaika sy ny couche sary — izany dia feno haingana amin'ny MikroTik. Tsy maintsy atodika amin'ny USB :

```routeros
/container/config/set \
  registry-url=https://registry-1.docker.io \
  tmpdir=usb1/tmp \
  layer-dir=usb1/layer
```

Hamarino :

```routeros
/container/config/print
```

## 4) Mamorona bridge ho an'ny container

```routeros
/interface/bridge/add name=dockers comment="Bridge containers Wima Zone"
/ip/address/add address=172.17.0.1/24 interface=dockers comment="Gateway bridge containers"
```

Ampio ny bridge ao amin'ny lisitra interface **LAN** mba ampiharina ny firewall sy DNS anatiny :

```routeros
/interface/list/member/add list=LAN interface=dockers
```

## 5) Mamorona VETH ho an'ny container billing

```routeros
/interface/veth/add name=veth-billing address=172.17.0.2/24 gateway=172.17.0.1 comment="Wima Zone"
/interface/bridge/port/add bridge=dockers interface=veth-billing
```

## 6) Ampio NAT mivoaka ho an'ny container

```routeros
/ip/firewall/nat/add chain=srcnat src-address=172.17.0.0/24 action=masquerade comment="Containers Internet"
```

## 7) Firewall (fanondranana portail + fidirana admin)

Atondro ny port ivelany **8080** an'ny routeur mankany amin'ny port 80 an'ny container (izay no hidiran'ny mpanjifa LAN amin'ny portail Wima Zone) :

```routeros
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8080 action=dst-nat \
  to-addresses=172.17.0.2 to-ports=80 comment="Fanondranana portail Wima Zone"
```

Omena alalana ny port miditra (chain `input`) :

```routeros
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8080 action=accept comment="Portail Wima Zone"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8291 action=accept comment="Winbox"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8728 action=accept comment="API MikroTik"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8729 action=accept comment="API-SSL MikroTik"
```

::: warning Container → RouterOS REST API (tsy maintsy ho an'ny licence)
Ny container dia manontany ny RouterOS amin'ny port **80** (HTTP) sy **443** (HTTPS) mba haka ny serial materially amin'ny alalan'ny REST API (jereo [fizarana 10 : licence](#_10-variables-tontolo-iainana-ho-an-ny-container)). Raha tsy ao amin'ny `interface-list=LAN` ny bridge `dockers` (na misy lalàna `drop !LAN` mihazona ny input), ampio mazava :

```routeros
/ip/firewall/filter/add chain=input action=accept \
  src-address=172.17.0.0/24 \
  protocol=tcp dst-port=80,443 \
  place-before=[find where chain=input action=drop in-interface-list="!LAN"] \
  comment="Container Wima Zone -> RouterOS REST API"
```

Raha tsy misy io lalàna io, tsy hahomby ny container amin'ny boot miaraka amin'ny `curl-error-7` na `curl-error-28` rehefa mamaky ny serial, dia `ERREUR: fingerprint invalide ou vide` — boot loop tsy mety mihatra.
:::

::: tip Laharan'ny lalàna
Apetraho ireo lalàna ireo **alohan'ny** `drop` ankapobeny amin'ny chain `input`. Raha tsy izany, tsy ampiasaina. Ampiasao `/ip/firewall/filter/move` raha ilaina, na ny `place-before=` etsy ambony izay manao izany ho azy.
:::

## 8) Apetraho ny DNS an'ny routeur

```routeros
/ip/dns/set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes
```

## 9) Mamorona fitehirizana MariaDB maharitra

Ny sary dia mitondra **MariaDB** ao anatiny ; tsy maintsy atao maharitra amin'ny USB ny lahatahiry MariaDB mba ho tafita amin'ny redémarrage / fanavaozana.

```routeros
/container/mounts/add src=usb1/billing-data dst=/data list=billing-db
```

::: danger `dst=` tsy maintsy `/data` (FA TSY `/var/lib/mysql`)
Ny sary dia mampiasa `/data/mysql` ho datadir MariaDB **ao anatin'ny** mount maharitra `/data`. Ity mount ity koa no mitahiry ny `/data/.fingerprint` (licence).

✅ OK : `src=usb1/billing-data` + `dst=/data`
❌ KO : `src=usb1/billing-data/mysql` + `dst=/var/lib/mysql` (tsy hita ny datadir → DB vaovao isaky ny pull, miverina ny migrations, very ny data)
:::

::: warning USB ext4 ilaina
Ny mount container dia tsy mandeha raha tsy amin'ny stockage voatsipika **ext4**. Hamarino amin'ny `/disk/print` fa hita ny `usb1`. Tsy handeha ny MariaDB amin'ny FAT32/NTFS.
:::

::: danger `src=` tsy maintsy ALOHAN'ny `root-dir`
Raha mampiasa `root-dir=usb1/wimazone` ianao, ny `src=` dia tokony hanondro lahatahiry **mpiray tampo** (oh. `usb1/billing-data`), **fa tsy sub-folder** ao anaty `root-dir`. Raha tsy izany dia voafafa ny mount rehefa `/container/remove` + repull — very ny MariaDB **sy** ny fingerprint licence, ka ho lavina HTTP 403 amin'ny boot manaraka.

✅ OK : `root-dir=usb1/wimazone` + `src=usb1/billing-data`
❌ KO : `root-dir=usb1/wimazone` + `src=usb1/wimazone/billing-data`
:::

## 10) Variables tontolo iainana ho an'ny container

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
/container/envs/add list=billing-env key=MIKROTIK_API_PASSWORD value=PASSWORD_ADMIN_ROUTEROS
/container/envs/add list=billing-env key=LARAVEL_AUTO_MIGRATION value=true
/container/envs/add list=billing-env key=LARAVEL_AUTO_MIGRATION_OPTIONS value=--force
/container/envs/add list=billing-env key=LARAVEL_AUTO_STORAGE_LINK value=true
/container/envs/add list=billing-env key=LARAVEL_ENABLE_QUEUE_WORKER value=true
/container/envs/add list=billing-env key=LARAVEL_ENABLE_SCHEDULER value=true
/container/envs/add list=billing-env key=LARAVEL_QUEUE_WORKER_OPTIONS value="--queue=mikrotik,default --tries=1 --timeout=1200 --sleep=2"
/container/envs/add list=billing-env key=REDIRECT_URL value=http://portail-anao.wifi
/container/envs/add list=billing-env key=HOTSPOT_STATUS_TIMEOUT_SECONDS value=2
/container/envs/add list=billing-env key=HOTSPOT_STATUS_CACHE_SECONDS value=3
/container/envs/add list=billing-env key=HOTSPOT_STATUS_FAILURE_COOLDOWN_SECONDS value=20
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC value=true
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC_ONLY_ONLINE value=true
/container/envs/add list=billing-env key=MIKROTIK_BOOT_HOTSPOT_SYNC_PROCESS_NOW value=false
```

::: info Licence
`WIMAZONE_LICENSE_KEY` dia alefa **amin'ny mail** an'ny ITDevSuccess rehefa avy mividy. Ampio ao amin'ny `billing-env` :

```routeros
/container/envs/add list=billing-env key=WIMAZONE_LICENSE_KEY value=<clé azo tamin'ny mail>
```

Manana clé manokana ny routeur tsirairay, azo foanana isaky ny iray amin'ny portail admin.
:::

::: tip Reverb WebSocket (cyber-café ihany)
Tsy alefa amin'ny default (`LARAVEL_ENABLE_REVERB=false`) satria tsy ilaina amin'ny hotspot tsotra. Ho an'ny **cyber-café** mila temps-réel (unlock haingana ny poste, monitoring sessions), alefaso :

```routeros
/container/envs/set [find list="billing-env" key="LARAVEL_ENABLE_REVERB"] value=true
# Avahao ny port WebSocket 8081 amin'ny LAN :
/ip/firewall/nat/add chain=dstnat protocol=tcp dst-port=8081 action=dst-nat \
  to-addresses=172.17.0.2 to-ports=8081 comment="Reverb WebSocket"
/ip/firewall/filter/add chain=input protocol=tcp dst-port=8081 action=accept comment="Reverb"
/container/stop [find name="Wima Zone"]
/container/start [find name="Wima Zone"]
```

Port 8081 (fa tsy 8080) mba hialana amin'ny disadisa amin'ny port portail captif.
:::

::: warning Identité matérielle anti-fraude
Ny licence dia **mifamatotra amin'ny serial materially** an'ny routeur (1 seat = 1 MikroTik). Amin'ny boot voalohany, ny container dia manontany ny RouterOS REST API (`https://172.17.0.1/rest/system/routerboard`) miaraka amin'ny `MIKROTIK_API_USER` / `MIKROTIK_API_PASSWORD` mba haka ny serial mivantana avy amin'ny matériel — **tsy azo amaivanana**.

Ny fingerprint alefa amin'ny serveur licence dia lasa `serial-<SN>`, **maharitra na avy amin'ny repull / reboot / réinstall RouterOS**. Raha misy routeur hafa miezaka mampiasa io licence io, ho lavina (HTTP 403).

::: tip Alefaso ny www service amin'ny MikroTik
```routeros
/ip/service/enable www
# Tsara raha voafetra amin'ny 172.17.0.0/24
/ip/service/set www address=172.17.0.0/24
```

⚠️ Tsy ampy ny manokatra ny service raha mihazona ny input avy amin'ny bridge container ny firewall. Jereo ny [lalàna `chain=input` ho an'ny `172.17.0.0/24` → port 80/443](#_7-firewall-fanondranana-portail-fidirana-admin) ao amin'ny fizarana 7.
:::
:::

## 11) Mamorona container Wima Zone

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

## 12) Manomboka ny container

```routeros
/container/start [find where name="Wima Zone"]
```

## 13) Hamarino ny log

```routeros
/container/log print follow where container="Wima Zone"
```

Haharetan'ny boot voalohany araka ny fitaovana :

| Modely | Boot voalohany | Reboot manaraka |
|---|---|---|
| hAP ax³ / RB5009 | 2-3 min | 30 s |
| L009 | 3-5 min | 45 s |

Tokony ho hitanao amin'ny farany :

```text
[scheduler] started
[queue] worker listening on mikrotik,default
[nginx] listening on 0.0.0.0:80
```

## 14) Walled Garden soso-kevitra {#walled-garden}

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

### Port fanampiny

```routeros
/ip/hotspot/walled-garden/add dst-port=8728 action=allow comment="API MikroTik"
/ip/hotspot/walled-garden/add dst-port=8291 action=allow comment="Winbox"
```

## 15) Fanamarinana aorian'ny fametrahana

```routeros
/interface/veth/print
/container/mounts/print
/container/envs/print
/ip/hotspot/walled-garden/print
/ip/hotspot/walled-garden/ip/print
/container/print
```

---

## <Icon name="LogIn" color="success" /> Fidirana voalohany {#premier-acces}

Rehefa mandeha ny container, mankanesa amin'ny portail admin :

**URL :** `http://172.17.0.2` (amin'ny fitaovana ao anatin'ny tambajotra anatiny).

**Porofo fidirana voalohany** (ovao avy hatrany) :

```text
Email    : admin@wimazone.local
Password : ChangeMe!2026
```

::: danger Asa voalohany tsy maintsy atao

1. Midira amin'ny super-admin.
2. Ovao ny tenimiafina avy amin'ny **Profil → Fiarovana**.
3. Apetraho ny API MVola / Befiana ao amin'ny **Paramètres → APIs**.
4. Ampio ny routeur MikroTik ao amin'ny **Paramètres → Routeurs**.
:::

## <Icon name="Database" color="primary" /> Backup sy famerenana {#backup-restore}

### Hitahiry ny MariaDB

Sokafy ny shell anatin'ny container avy eo avoay amin'ny `mysqldump` :

```routeros
/container/shell [find where name="Wima Zone"]
```

Avy eo ao anatin'ny shell :

```bash
mysqldump -u root -p"$MARIADB_ROOT_PASSWORD" wimazone > /data/mysql/backups/wimazone-$(date +%F).sql
```

Ny rakitra `.sql` dia hita avy amin'ny RouterOS ao amin'ny `usb1/billing-data/mysql/backups/`. Raha te-hampiasa lahatahiry backup manokana :

```routeros
/file/copy src=usb1/billing-data/mysql/backups/wimazone-YYYY-MM-DD.sql dst=usb1/backup/wimazone-YYYY-MM-DD.sql
```

Na alaina avy amin'ny host lavitra amin'ny SCP :

```bash
scp admin@192.168.88.1:/usb1/backup/wimazone-2026-04-24.sql ./
```

### Hamerina backup

```routeros
/container/shell [find where name="Wima Zone"]
```

Ao anatin'ny shell :

```bash
mysql -u root -p"$MARIADB_ROOT_PASSWORD" wimazone < /data/mysql/backups/wimazone-YYYY-MM-DD.sql
```

::: tip Automatique
Ampio scheduler RouterOS manao dump isan'andro :

```routeros
/system/scheduler/add name=billing-backup interval=1d start-time=03:00 on-event={
  /container/shell [find where name="Wima Zone"] command="sh -c 'mysqldump -u root -p\"\$MARIADB_ROOT_PASSWORD\" wimazone > /data/mysql/backups/wimazone-daily.sql'"
}
```

:::

## <Icon name="RefreshCw" color="info" /> Fanavaozana {#mise-a-jour}

Ny fanavaozana dia dingana roa : alaina ny sary vaovao avy eo averina forona ny container. Ny migration Laravel dia mandeha ho azy rehefa miverina ny container (`LARAVEL_AUTO_MIGRATION=true`).

```routeros
# 1. Ajanona ary esory ny container ankehitriny (ny data amin'ny USB tsy voakasika)
/container/stop [find where name="Wima Zone"]
/container/remove [find where name="Wima Zone"]

# 2. Averina forona amin'ny configuration mitovy
/container/add \
  name="Wima Zone" \
  remote-image=wimazone/billing:latest \
  interface=veth-billing \
  root-dir=usb1/wimazone \
  mounts=billing-db \
  envlist=billing-env \
  start-on-boot=yes \
  logging=yes

# 3. Alefa
/container/start [find where name="Wima Zone"]
```

::: warning Mitahiry aloha
Manao dump MariaDB (jereo ny fizarana eo aloha) **alohan'ny** fanavaozana.
:::

## <Icon name="Wrench" color="warning" /> Famahana olana {#depannage}

### Tsy mandeha ny container

```routeros
/container/log print where container="Wima Zone"
```

Hafatra mahazatra :

| Hafatra | Antony | Vahaolana |
|---|---|---|
| `exited with signal 4 (Illegal instruction)` | Routeur misy CPU EN7562CT (hEX refresh / hEX S 2025) manambara `archVariant:v5` → sandbox MikroTik voafetra amin'ny arm32v5 soft-float, tsy mifanaraka amin'ny Alpine armhf | Tsy tohanana ireo modely ireo. Ampiasao L009, hAP ax³ na RB5009 |
| `SIGKILL` / `OOMKilled` | Tsy ampy RAM | Ahena ny queue workers, ampiasao ax³ |
| `licence rejetee (HTTP 401/403)` | Licence diso, voafoana, na seat efa azon'ny fingerprint hafa | Jereo fizarana [Repull namafa /data](#repull-fingerprint-very) etsy ambany |
| `impossible de lire le serial via RouterOS REST API` | `MIKROTIK_API_*` diso na tsy mandeha ny service `www` | `/ip/service/enable www` + hamarino user/password admin RouterOS |
| `Can't connect to MySQL server on '127.0.0.1'` | Mbola tsy vonona ny MariaDB | Miandrasa 30 s aorian'ny boot ; jereo `s6-svstat mariadb` |
| `Access denied for user 'wimazone'` | Tenimiafina DB diso | Hamarino `DB_PASSWORD` sy `MARIADB_ROOT_PASSWORD` |
| `Unknown database 'wimazone'` | Mount `/data` foana na simba | Hamarino `src=usb1/billing-data dst=/data`, foronina indray ny mount |
| Miverina daholo ny migrations + DB vaovao isaky ny pull | Mount mampiasa `dst=` diso (oh. `/var/lib/mysql`) — tsy maharitra mihitsy ny datadir MariaDB | Foronina indray ny mount amin'ny `dst=/data` (jereo section 9) |
| `502 Bad Gateway` | PHP-FPM feno | Ahena `HOTSPOT_STATUS_TIMEOUT_SECONDS=1` |
| `max_children reached` | Fangatahana be loatra | Hajao `MIKROTIK_BOOT_HOTSPOT_SYNC_PROCESS_NOW=false` |

### Repull namafa /data — licence lavina HTTP 403 {#repull-fingerprint-very}

**Soritr'aretina** amin'ny boot aorian'ny `/container/remove` + repull :

```
[startup] ERREUR: licence rejetee (HTTP 403) — verifier WIMAZONE_LICENSE_KEY
[startup] ERREUR: licence invalide — pas de fallback offline pour ce cas.
```

**Antony** : tsy maharitra tsara ny mount `/data` (jereo [warning fizarana 9](#9-mamorona-fitehirizana-mariadb-maharitra)). Very ny `/data/.fingerprint`. Raha ny fingerprint taloha dia UUID kisendrasendra (client taloha < v4.10.2), mbola mihazona ny seat amin'ny serveur izany → lavina ny vaovao.

**Vahaolana** (atao indray mandeha) :

1. **Amin'ny wimazone.mg/admin/licenses** : tadiavo ny licence, fantaro ny usage taloha (`uuid-...`), tsindrio **« Libérer »** mba hanafoanana ny seat.
2. **Amin'ny MikroTik**, hamarino fa ny credentials API dia napetraka :

   ```routeros
   /ip/service/enable www
   /container envs add list=billing-env key=MIKROTIK_API_USER value=admin
   /container envs add list=billing-env key=MIKROTIK_API_PASSWORD value=<pass>
   /container stop  [find name=billing]
   /container start [find name=billing]
   ```

3. **Amin'ny boot manaraka**, ny container dia maka ny serial mivantana avy amin'ny REST API ary mametraka azy amin'ny serveur. **Maharitra na avy amin'ny repull, reboot, ary na dia réinstall RouterOS aza** io identité matérielle io — tsy hiverina intsony ny olana.

4. **Hahitsy ny mount** mba hahatafita koa ny MariaDB :

   ```routeros
   /container/mounts/add src=usb1/billing-data dst=/data list=billing-db
   # src= ALOHAN'ny root-dir (jereo fizarana 9)
   ```

### Diagnostic tambajotra (VETH / bridge)

```routeros
/interface/veth/print detail
/interface/bridge/port/print where bridge=dockers
/ip/address/print where interface=dockers
/ping 172.17.0.2 count=4
```

Raha tsy mamaly ny `172.17.0.2`, hamarino fa tafaray amin'ny VETH ny container :

```routeros
/container/print detail
```

### Sync hotspot mitsahatra

```routeros
/container/shell [find where name="Wima Zone"]
# Ao anatin'ny container :
php artisan hotspot:sync --router=1 --dry-run
```

### Log Laravel

Avy amin'ny shell anatin'ny container :

```bash
tail -f /var/www/html/storage/logs/laravel.log
```

### API MikroTik tsy azo idirana

Hamarino ny credentials sy ny port :

```routeros
/user/print
/ip/service/print
# API tokony alefa (port 8728) na API-SSL (8729)
/ip/service/enable api
```

---

## <Icon name="BookOpen" color="info" /> Notes fampandehanana

- `OFFLINE_FALLBACK=true` : ny container dia manomboka amin'ny kaody eto an-toerana raha tsy tafiditra ny API wimazone.
- Mount `/data` amin'ny `usb1/billing-data` : maharitra ny data MariaDB (`/data/mysql`) sy ny fingerprint licence (`/data/.fingerprint`) na dia averina forona aza ny container.
- `LARAVEL_AUTO_STORAGE_LINK=false` : misoroka olana amin'ny mount sasany.
- Atao mialoha ny assets frontend (tsy atao amin'ny MikroTik ny build mavesatra).
- Ho an'ny fanaraha-maso : `/container/log print follow where container="Wima Zone"`.
