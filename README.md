# PlusEMU hotel in Docker

Draait een PlusEMU-hotel (emulator, Volt-client en Atom CMS) in Docker, met MariaDB.
Gebaseerd op [plusemu-installer](https://github.com/plus-emulator/plusemu-installer): dezelfde onderdelen, maar in containers.

## Onderdelen

| Service | Wat |
|---|---|
| `db` | MariaDB 11.4 (poort 3306 is niet gepubliceerd) |
| `hotel` | PlusEMU + nginx + PHP 8.5-FPM + Atom CMS + Volt-client, gestuurd door supervisord |
| `cloudflared` | Optioneel: Cloudflare Tunnel in dezelfde stack (`--profile tunnel`) |

Bestanden: `docker-compose.yml`, `.env.example`, `hotel/` (Dockerfile, entrypoint.sh, nginx.conf, supervisord.conf) en `plusemu-hotel-build.tar.gz` (dezelfde `hotel/`-map als archief, voor Portainer).

## Vereisten

- Docker met Compose (of Portainer)
- ca. 3 GB vrije schijfruimte en internet bij het bouwen en de eerste start
- Een domein bij Cloudflare met een Tunnel (of een andere proxy die HTTPS regelt). De container zelf praat alleen HTTP op poort 80.

## Installeren

**1. De image bouwen** (eenmalig, duurt enkele minuten)

- Terminal: `docker build -t plusemu-hotel:latest ./hotel`
- Portainer: *Images → Build a new image*, naam `plusemu-hotel:latest`, kies *Upload* en selecteer `plusemu-hotel-build.tar.gz`.

**2. Variabelen instellen** (kopieer `.env.example` naar `.env`, of vul ze in bij *Environment variables* in Portainer)

| Variabele | Betekenis |
|---|---|
| `DOMAIN` | Je hotel-adres, zonder `https://` (bijv. `hotel.jouwdomein.nl`) |
| `WS_URL` | Leeg laten = `wss://<DOMAIN>/ws` |
| `HOTEL_NAME` | Naam van het hotel |
| `DB_PASSWORD`, `DB_ROOT_PASSWORD` | Kies lange wachtwoorden met alleen letters en cijfers, **voor de eerste start**. Later wijzigen werkt niet meer zonder het databasevolume te verwijderen. |
| `HOTEL_PORT` | Poort op de server (standaard 8085) |
| `TUNNEL_TOKEN` | Alleen voor het `cloudflared`-profiel |

**3. De stack starten**

- Terminal: `docker compose up -d`
- Portainer: *Stacks → Add stack → Upload* met `docker-compose.yml`, daarna *Deploy*.

De eerste start downloadt de hotel-files (ca. 550 MB) en importeert de database. Volg dit met `docker compose logs -f hotel`.

**4. Cloudflare Tunnel koppelen**

Voeg bij je tunnel een *Public hostname* toe voor je `DOMAIN`:

- Tunnel in dezelfde stack (`docker compose --profile tunnel up -d`): service `http://hotel:80`
- Bestaande tunnel elders: service `http://<server-ip>:8085`

WebSockets moeten aanstaan in Cloudflare (standaard het geval).

## Installeren met Portainer

Portainer kan geen map bouwen vanuit een compose-bestand. Daarom bouw je eerst de image en rol je daarna de stack uit.

**Stap 1: de image bouwen**

1. Ga naar **Images → Build a new image**.
2. Vul bij *Name* `plusemu-hotel:latest` in.
3. Kies **Upload** en selecteer `plusemu-hotel-build.tar.gz`. Selecteer geen zip, want dat werkt niet.
4. Klik op **Build the image** en wacht tot het klaar is (enkele minuten). Je ziet de build-output op het scherm.
5. Controleer onder **Images** dat `plusemu-hotel:latest` in de lijst staat.

**Stap 2: de stack uitrollen**

1. Ga naar **Stacks → Add stack** en geef de stack een naam, bijvoorbeeld `plusemu-hotel`.
2. Kies **Upload** en selecteer `docker-compose.yml` (niet de zip, dat geeft een YAML-fout).
3. Vul bij **Environment variables** de variabelen uit de tabel hierboven in. Met *Load variables from .env file* kun je `.env.example` inladen. Pas daarna `DOMAIN`, `DB_PASSWORD` en `DB_ROOT_PASSWORD` aan.
4. Wil je de tunnel in dezelfde stack draaien? Voeg dan ook `COMPOSE_PROFILES` = `tunnel` en `TUNNEL_TOKEN` toe.
5. Klik op **Deploy the stack**.

**Stap 3: controleren**

- Ga naar **Containers**. Na enkele minuten staan `…-db-1` en `…-hotel-1` op *healthy*. De eerste start duurt lang door de download van de hotel-files.
- Klik op de hotelcontainer en dan op **Logs** om de voortgang te zien.
- Een console open je bij de container met het icoon `>_` (kies `/bin/bash`). Daar voer je de commando's van "De CMS instellen" uit.

**Bijwerken in Portainer**

1. Bouw de image opnieuw met stap 1 (zelfde naam en tag).
2. Open de stack, klik op **Editor** en daarna op **Update the stack**. Zet *Re-pull image* uit, want de image staat alleen lokaal.

**Bij problemen in Portainer**

- `pull access denied for plusemu-hotel`: de image staat niet op deze Docker-host. Controleer stap 1 en kijk linksboven of je in dezelfde *Environment* zit als bij de stack.
- `YAMLSyntaxError`: je hebt de zip geüpload in plaats van `docker-compose.yml`.
- Wijzig je `DB_PASSWORD` na de eerste start, dan werkt de database niet meer. Verwijder dan de stack en het volume `…_db` (onder **Volumes**) en rol opnieuw uit. Alle data in de database gaat dan verloren.

## De CMS instellen (eenmalig)

Open een console in de hotelcontainer (`docker exec -it <stacknaam>-hotel-1 bash`, of Portainer → Container → Console).

```
cd /opt/hotel/cms
mkdir -p storage/backups
mariadb-dump --skip-ssl -h db -uhotel -p plus > storage/backups/start.sql   # backup, typ DB_PASSWORD
runuser -u www-data -- php artisan atom:install --emulator=plus --theme=atom --skip-build --no-interaction
runuser -u www-data -- php artisan optimize:clear
chown -R www-data:www-data storage bootstrap/cache
```

De container zet zelf `EMULATOR_DRIVER=plus`, de RCON-poort, de client-URL en de paden in `website_settings`. Gebruik **nooit** `--fresh`, dat wist alle tabellen.

Daarna kan de wizard op `https://<DOMAIN>/installation` om de installatiesleutel vragen:

```
mariadb --skip-ssl -h db -uhotel -p plus -e "SELECT installation_key FROM website_installation;"
```

### Wachtwoord van een account opnieuw instellen

```
cd /opt/hotel/cms
read -rsp 'Nieuw wachtwoord: ' NEWPW; echo
runuser -u www-data -- env ADMIN_USER=Admin ADMIN_PASSWORD="$NEWPW" php <<'PHP'
<?php
require 'vendor/autoload.php';
$app = require 'bootstrap/app.php';
$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();
$user = App\Models\User::where('username', getenv('ADMIN_USER'))->firstOrFail();
$user->forceFill(['password' => getenv('ADMIN_PASSWORD')])->save();
echo "Wachtwoord gewijzigd\n";
PHP
unset NEWPW
```

Wachtwoorden worden gehasht opgeslagen; teruglezen kan niet. Minimaal 12 tekens met hoofdletter, kleine letter, cijfer en symbool.

## Dagelijks gebruik

- Logs: `docker compose logs -f hotel`
- Database openen: `docker exec -it <stacknaam>-db-1 mariadb -uhotel -p plus`
- Van buiten: SSH-tunnel (`ssh -N -L 3307:<db-container-ip>:3306 gebruiker@server`), poort 3306 nooit publiek maken.
- Bijwerken: image opnieuw bouwen en de stack opnieuw uitrollen. Database, config, hotel-files en CMS-opslag blijven in volumes (`db`, `emu-config`, `hotel-files`, `cms-storage`).
- Backup: maak regelmatig een dump met `mariadb-dump` (zie boven) en kopieer die van de server af.

## Problemen oplossen

| Melding | Oorzaak en oplossing |
|---|---|
| `pull access denied for plusemu-hotel` | De image is niet gebouwd of staat op een andere Docker-host. Voer stap 1 opnieuw uit en controleer de naam en tag. |
| 500-pagina | `tail -n 40 /opt/hotel/cms/storage/logs/laravel.log` in de container. |
| `Vite manifest not found` | De front-endbuild ontbreekt. Bouw de image opnieuw (thema in `ATOM_THEME`, standaard `atom`). |
| `Unknown column 'ip'` in `bans` | De CMS draait op de verkeerde emulatordriver. Controleer `EMULATOR_DRIVER="plus"` in `/opt/hotel/cms/storage/.env`. |
| Poorten of adres fout na wijziging van `DOMAIN` | De stack opnieuw uitrollen; de client-config wordt bij het starten geschreven. |

## Status en beperkingen

- Getest: image bouwt, containers worden healthy, de site bereikbaar via de tunnel, de installatiewizard doorlopen.
- Niet bevestigd: `atom:install` met de Plus-driver op een bestaande database, de beheerdersrechten voor housekeeping en het daadwerkelijk inloggen in het spel.
- De camera-service (foto's in het hotel) is niet meegenomen.
- Het Dockerfile haalt de nieuwste PlusEMU-, Volt- en Atom-versie op tijdens het bouwen. Een latere build kan dus een nieuwere versie bevatten.
