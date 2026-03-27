# ARR Stack Homelab Setup

> **Gebaseerd op het werk van [automation-avenue](https://github.com/automation-avenue/arr-new) — New ARR Stack 2026**
> Met dank aan hun docker-compose configuratie en uitgebreide documentatie.

---

> ⚠️ **Disclaimer**
> Deze handleiding is uitsluitend bedoeld voor informatieve doeleinden. Het downloaden van auteursrechtelijk beschermd materiaal zonder toestemming van de rechthebbende is in veel landen **illegaal**. De auteur is niet verantwoordelijk voor enig misbruik van de hier beschreven software of technieken. Zorg ervoor dat je altijd de wet- en regelgeving in jouw land naleeft en gebruik deze tools alleen voor legale doeleinden.

---

Deze handleiding beschrijft hoe je een volledige ARR media-automatiseringsstack opzet met Docker.
De instructies zijn geschreven voor Debian/Ubuntu, maar Docker draait op elke Linux-distributie.
Gebruik je Windows of macOS? Dan kun je [Docker Desktop](https://docs.docker.com/desktop/) gebruiken.

---

## Docker installeren en omgeving voorbereiden

Installeer Docker en Docker Compose via de [officiële documentatie](https://docs.docker.com/compose/).
Ga naar **Install → Plugin → Install using the Repository → Ubuntu** (of jouw OS), of ga direct naar [deze pagina](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

Voer de commando's uit die daar staan, vergelijkbaar met:

```bash
# Voeg Docker's officiële GPG-sleutel toe:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Voeg de repository toe aan de apt-bronnen:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Installeer daarna de Docker-pakketten en controleer of alles werkt:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run hello-world
```

Controleer of Docker Compose correct is geïnstalleerd:

```bash
docker compose version
```

### Mappenstructuur aanmaken

Maak de aanbevolen mappenstructuur aan volgens de [TRaSH Guide](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/):

```bash
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}
sudo apt install tree
tree /data
sudo chown -R 1000:1000 /data
sudo chmod -R a=,a+rX,u+w,g+w /data
ls -ln /data
```

> Gebruik je naast torrents ook een Usenet-client (zoals NZBGet of SABnzbd)? Vervang dan de eerste regel door:
> ```bash
> mkdir -p /data/{usenet/{incomplete,complete}/{tv,movies,music},media/{tv,movies,music}}
> ```

### Docker Compose bestand

De `docker-compose.yml` van dit project is beschikbaar op [GitHub](https://github.com/automation-avenue/arr-new/blob/main/docker-compose.yml).
Kloon de repository of kopieer het bestand handmatig:

```bash
git clone https://github.com/automation-avenue/arr-new.git
# of maak het bestand handmatig aan:
sudo nano docker-compose.yml
```

> Hostnamen zijn niet nodig in dit configuratiebestand, omdat alle containers op hetzelfde Docker-netwerk draaien.

---

## Eerste keer opstarten

Start alle services met één commando:

```bash
sudo docker compose up -d
```

---

## Services configureren

Na het opstarten moeten de interne instellingen van elke applicatie op elkaar worden afgestemd:

| Service | Poort | Functie |
|---------|-------|---------|
| qBittorrent | `8080` | Torrent download client |
| Prowlarr | `9696` | Indexer beheer |
| Radarr | `7878` | Films automatisering |
| Sonarr | `8989` | Series automatisering |
| Lidarr | `8686` | Muziek automatisering |
| Bazarr | `6767` | Ondertitels |
| Plex | `32400` | Media afspelen |
| Seerr | `5055` | Media requests |
| Profilarr | `6868` | Kwaliteitsprofielen |
| Tdarr | `8265` | Automatisch transcoden |

Omdat alle paden onder dezelfde mount (`/data`) vallen, behandelt het OS ze als één bestandssysteem. Dit maakt instant hardlinks (ook wel atomic moves) mogelijk.

---

### qBittorrent

Controleer de tijdelijke inloggegevens in de container-logs:

```bash
sudo docker logs qbittorrent
```

Je ziet iets als:
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: <jouw-wachtwoord>
```

Open de webinterface via:
- Op de host zelf: `http://localhost:8080`
- Vanaf een ander apparaat: `http://<host-ip>:8080`

Wijzig het wachtwoord via **Tools → Options → WebUI**.

#### Categorieën aanmaken

Ga naar het linkerpaneel → **Categories → All → rechtermuisknop → Add category**:

| Service | Categorie | Pad      |
|---------|-----------|----------|
| Radarr  | `movies`  | `movies` |
| Sonarr  | `tv`      | `tv`     |
| Lidarr  | `music`   | `music`  |

> Maak de categorieën aan **vóórdat** je de overige instellingen configureert, anders kunnen ze verdwijnen.

#### Downloadinstellingen

Ga naar **Tools → Options → Downloads → Saving Management** en stel het volgende in (zie ook de [TRaSH Guide](https://trash-guides.info/Downloaders/qBittorrent/How-to-add-categories/)):

- **Default Torrent Management Mode**: `Automatic`
- **When Torrent Category changed**: `Relocate torrent`
- **When Default Save Path Changed**: `Switch affected torrents to Manual Mode`
- **When Category Save Path Changed**: `Switch affected torrents to Manual Mode`
- Vink **beide** vakjes aan: `Use Subcategories` en `Use Category paths in Manual Mode`
- **Default Save Path**: `/data/torrents`

Sla op met de **Save**-knop onderaan.

> Heb je problemen met categorieën? Probeer dan een alternatieve qBittorrent-image:
> ```yaml
>   qbittorrent:
>     <<: *common-keys
>     container_name: qbittorrent
>     image: ghcr.io/qbittorrent/docker-qbittorrent-nox:latest
>     ports:
>       - 8080:8080
>       - 6881:6881
>       - 6881:6881/udp
>     environment:
>       - QBT_LEGAL_NOTICE=confirm
>       - WEBUI_PORT=8080
>       - TORRENTING_PORT=6881
>     volumes:
>       - /etc/localtime:/etc/localtime:ro
>       - /docker/appdata/qbittorrent:/config
>       - /data:/data
> ```

---

### Prowlarr

Open: `http://<host-ip>:9696`

Stel bij het eerste gebruik een gebruikersnaam en wachtwoord in via **Form (login page) authentication**.

Ga naar **Settings → Download Clients → +** en voeg qBittorrent toe:
- **Use SSL**: uitvinken
- **Host**: `qbittorrent`
- **Port**: `8080`
- **Username/Password**: de inloggegevens die je in qBittorrent hebt ingesteld

Test de verbinding en sla op.

---

### Radarr

Open: `http://<host-ip>:7878`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/movies`
2. **Settings → Media Management → Show Advanced → Importing**: zet **Use Hardlinks instead of Copy** aan

Optioneel: zet **Rename Movies**, **Delete empty movie folders during disk scan** en **Import Extra Files** (`srt,sub,nfo`) aan.

3. **Settings → Download Clients → +**: voeg qBittorrent toe (zelfde stappen als bij Prowlarr), maar stel de categorie in op `movies`
4. **Settings → General**: kopieer de API-sleutel
5. Ga naar **Prowlarr → Settings → Apps → + → Radarr**: plak de API-sleutel
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Radarr Server**: `http://radarr:7878`

Test en sla op.

---

### Sonarr

Open: `http://<host-ip>:8989`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/tv`
2. Zet **Use Hardlinks instead of Copy** aan
3. Optioneel: **Rename Episodes**, **Delete empty Folders**, **Import Extra Files** (`srt,sub,nfo`)
4. Voeg qBittorrent toe als download client, categorie: `tv` (standaard staat dit op `tv-sonarr`, pas dit aan)
5. Koppel aan Prowlarr via **Settings → Apps → + → Sonarr**:
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Sonarr Server**: `http://sonarr:8989`

---

### Lidarr

Open: `http://<host-ip>:8686`

1. **Settings → Media Management → Add Root Folder**: stel in op `/data/media/music`
2. Voeg qBittorrent toe als download client, categorie: `music` (standaard: `lidarr`)
3. Koppel aan Prowlarr:
   - **Prowlarr Server**: `http://prowlarr:9696`
   - **Lidarr Server**: `http://lidarr:8686`

---

### Bazarr

Open: `http://<host-ip>:6767`

- **Settings → Languages**: maak een taalprofiel aan (bijv. "English" of "Any")
- **Settings → Providers**: voeg ondertitelbronnen toe zoals OpenSubtitles.org (gratis account vereist)
- Ga na het koppelen van Radarr/Sonarr naar het **Series** of **Movies** tabblad en klik **Update** om je bibliotheek te synchroniseren

---

### Seerr

Open: `http://<host-ip>:5055`

Seerr is de opvolger van Jellyseerr en Overseerr en biedt een nette interface waarmee vrienden en familie zelf media kunnen aanvragen — zonder toegang tot Radarr of Sonarr.

1. Maak een beheerdersaccount aan bij het eerste opstarten
2. Ga naar **Settings → Plex** en koppel je Plex-account, of kies **Jellyfin** als je die gebruikt
3. Ga naar **Settings → Radarr** en voeg Radarr toe:
   - **Host**: `radarr`
   - **Port**: `7878`
   - **API Key**: kopieer uit Radarr → Settings → General
   - Selecteer je gewenste kwaliteitsprofiel en rootmap
4. Herhaal stap 3 voor **Settings → Sonarr** met host `sonarr` en poort `8989`

Gebruikers kunnen nu via Seerr films en series aanvragen die automatisch worden opgepakt door Radarr en Sonarr.

---

### Profilarr

Open: `http://<host-ip>:6868`

Profilarr synchroniseert automatisch kwaliteitsprofielen en aangepaste formaten naar Radarr en Sonarr op basis van de TRaSH Guides.

1. Ga naar **Settings → Radarr** en voeg Radarr toe:
   - **Host**: `http://radarr:7878`
   - **API Key**: kopieer uit Radarr → Settings → General
2. Herhaal voor **Settings → Sonarr** met `http://sonarr:8989`
3. Ga naar **Config** en kies welke TRaSH-profielen je wil synchroniseren
4. Klik op **Sync** om de profielen direct toe te passen

> ⚠️ Profilarr overschrijft bestaande kwaliteitsprofielen. Maak eerst een backup via **Settings → Backup** in Radarr/Sonarr als je bestaande profielen wil bewaren.

---

### Tdarr

Open: `http://<host-ip>:8265`

Tdarr transcodeert automatisch je mediabestanden naar efficiëntere formaten (bijv. H.265/HEVC) om schijfruimte te besparen.

Maak eerst de tijdelijke transcodeermap aan op je host:

```bash
sudo mkdir -p /transcode_cache
sudo chown -R 1000:1000 /transcode_cache
```

Configuratie in de webinterface:

1. Ga naar **Libraries → +** en voeg je mediamap toe:
   - **Source**: `/data/media/movies` (voeg ook `/data/media/tv` toe als aparte library)
   - **Transcode cache**: `/temp`
2. Ga naar **Transcode options** en kies je gewenste codec:
   - `hevc_nvenc` — NVIDIA GPU
   - `hevc_vaapi` — Intel/AMD GPU
   - `libx265` — CPU (zwaar, maar werkt altijd)
3. Stel een **Health Check** in om beschadigde bestanden op te sporen
4. Zet de library op **Enabled** om te starten

> ⚠️ CPU-transcoding (`libx265`) is zwaar. Start met een kleine testmap voordat je je volledige bibliotheek laat transcoden.

---

## Services herstarten

Start alle services opnieuw om te controleren of alles correct opstart:

```bash
sudo docker compose down
sudo docker compose up -d
```

> De melding `WARN[0000] No services to build` is normaal en kan worden genegeerd.

---

## Indexers toevoegen

> ⚠️ **Belangrijk: gebruik alleen legale indexers**
> Het gebruik van indexers voor het downloaden van auteursrechtelijk beschermd materiaal is in veel landen strafbaar en kan leiden tot juridische consequenties. Informeer jezelf altijd over de wetgeving in jouw land voordat je indexers toevoegt. De auteur van deze handleiding draagt geen verantwoordelijkheid voor hoe je deze tools gebruikt.

Nu kun je indexers toevoegen via Prowlarr. Zoek online naar legale indexers zoals:

- **The Internet Archive (archive.org)** — duizenden publieksdomeinefilms en -series
- Radarr ondersteunt films in het publiek domein of onder Creative Commons-licentie, zoals *Night of the Living Dead (1968)*, *His Girl Friday (1940)* en *Charade (1963)*

> De ARR-stack is niet uitsluitend bedoeld voor auteursrechtelijk beschermd materiaal. Het zijn krachtige automatiseringstools die ook uitstekend werken met legale, vrij beschikbare content.

---

## Media Players

Voor het afspelen van je mediabibliotheek kun je kiezen tussen **Jellyfin** (volledig gratis en open-source) of **Plex** (gratis met optionele betaalde Plex Pass). Beide werken uitstekend met de ARR-stack.

---

### Jellyfin

Open: `http://<host-ip>:8096`

Maak een gebruikersaccount aan en voeg mediabibliotheken toe:

- **Movies**: koppel aan `/data/media/movies`
- **TV Shows**: koppel aan `/data/media/tv`
- **Music**: koppel aan `/data/media/music`

Voeg Jellyfin toe aan je `docker-compose.yml`:

```yaml
  jellyfin:
    <<: *common-keys
    container_name: jellyfin
    image: ghcr.io/hotio/jellyfin:latest
    ports:
      - 8096:8096
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/jellyfin:/config
      - /data/media:/data/media
```

> **Voordelen Jellyfin**: volledig gratis, geen account vereist, geen limieten, open-source en zelf gehost.

---

### Plex

Open: `http://<host-ip>:32400/web`

Plex vereist een gratis Plex-account op [plex.tv](https://www.plex.tv). Haal eerst een **claim token** op via `https://www.plex.tv/claim` — dit token is 4 minuten geldig en koppelt de server aan jouw account.

Voeg Plex toe aan je `docker-compose.yml`:

```yaml
  plex:
    <<: *common-keys
    container_name: plex
    image: ghcr.io/hotio/plex:latest
    ports:
      - 32400:32400
    environment:
      - PLEX_CLAIM=claim-xxxxxxxxxxxxxxxxxxxx  # vervang door jouw token van plex.tv/claim
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/plex:/config
      - /data/media:/data/media
```

Na het opstarten log je in via de webinterface en voeg je mediabibliotheken toe:

- **Movies**: koppel aan `/data/media/movies`
- **TV Shows**: koppel aan `/data/media/tv`
- **Music**: koppel aan `/data/media/music`

> **Voordelen Plex**: uitgebreide apps voor vrijwel elk platform, sterk ecosystem en goede remote access. Sommige functies (zoals offline downloads en geavanceerde audioprofielen) vereisen een betaalde **Plex Pass**.

## Probleemoplossing

### DNS controleren

Controleer of containers de Cloudflare DNS gebruiken (geconfigureerd in `docker-compose.yml`):

```bash
sudo docker exec -it radarr cat /etc/resolv.conf
```

### Hardlinks controleren

Navigeer naar `/data` en controleer de mappenstructuur:

```bash
tree /data
du -sch *
```

Vergelijk de inode-nummers van hetzelfde bestand in zowel de torrent- als mediamappen:

```bash
ls -i /data/media/movies/<bestandsnaam>
ls -i /data/torrents/movies/<bestandsnaam>
```

Als de inode-nummers overeenkomen, werken de hardlinks correct. Werkt het niet? Controleer dan de logbestanden (**System → Log Files** in Radarr/Sonarr) — rechten op bron of bestemming zijn de meest voorkomende oorzaak.

### Bestanden verplaatsen niet automatisch

Als bestanden niet automatisch van de torrentmap naar de mediamap worden verplaatst, ga dan naar **Activity → Queue**. Zie je de melding *Downloaded - Unable to Import Automatically*? Klik dan op het handmatig importeer-icoon (het hoofd-icoontje rechts in de rij) en bevestig de juiste film of serie.

---

## Optionele uitbreidingen

### FlareSolverr (Cloudflare bypass)

Voeg toe aan je `docker-compose.yml` als Prowlarr bepaalde sites niet kan indexeren door Cloudflare-blokkades:

```yaml
  flaresolverr:
    <<: *common-keys
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports:
      - 8191:8191
    environment:
      - LOG_LEVEL=info
```

Configureer daarna in Prowlarr via **Settings → Indexers → + → FlareSolverr**:
- **Host**: `http://flaresolverr:8191`
- **Tags**: `cloudflare`

### Jellyfin hardwareversnelling

Voeg de volgende regels toe aan de Jellyfin-service in `docker-compose.yml` om GPU-versnelling in te schakelen (vereist aanvullende configuratie buiten Docker):

```yaml
jellyfin:
    <<: *common-keys
    devices:
      - /dev/dri:/dev/dri
```

### Plex hardwareversnelling

Voor Plex werkt hardwareversnelling op dezelfde manier. Voeg dit toe aan de Plex-service in `docker-compose.yml`:

```yaml
plex:
    <<: *common-keys
    devices:
      - /dev/dri:/dev/dri
```

> Let op: hardwaretranscodering in Plex vereist een actieve **Plex Pass**.

### SABnzbd (Usenet-client)

Voeg toe aan `docker-compose.yml` als je SABnzbd gebruikt in plaats van of naast qBittorrent:

```yaml
  sabnzbd:
    container_name: sabnzbd
    image: ghcr.io/hotio/sabnzbd:latest
    ports:
      - 8080:8080
      - 9090:9090
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/sabnzbd:/config
      - /data:/data
```

> Draai je zowel qBittorrent als SABnzbd? Dan botsen ze op poort `8080`. Pas de externe poort van één van de services aan, bijvoorbeeld:
> ```yaml
>     ports:
>       - 8081:8080
> ```

Voor de SABnzbd-configuratie, zie de [TRaSH Guide voor mappenstructuur](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/) en de [SABnzbd basisinstellingen](https://trash-guides.info/Downloaders/SABnzbd/Basic-Setup/).

---

## Credits

Deze handleiding is gebaseerd op en geïnspireerd door:

- **[automation-avenue/arr-new](https://github.com/automation-avenue/arr-new)** — New ARR Stack 2026
- **[TRaSH Guides](https://trash-guides.info/)** — voor mappenstructuur, hardlink-configuratie en downloadclientinstellingen
- **[Servarr Wiki](https://wiki.servarr.com/)** — voor Radarr, Sonarr en aanverwante documentatie
